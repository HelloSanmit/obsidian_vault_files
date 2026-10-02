
# Phase 8 — Lesson 30: Generic Constraints — “Any Type” Is Too Much Freedom

Last lesson, we learned this:

```kotlin
fun <T> something(value: T)
```

which basically means:

> "I don't care what `T` is."

That's useful.

But now imagine I write:

```kotlin
fun <T> printLength(value: T) {
    println(value.length)
}
```

Kotlin should object.

Why?

Because `T` could be:

```text
String
Int
Boolean
User
Dog
Banana
```

and most of those don't have a `length`.

So we've reached an important problem:

> **I want generic code, but I don't want completely unrestricted generic code.**

I want to say:

> "Give me any type you like, as long as that type satisfies a particular rule."

That's what **generic constraints** are for.

---

# Let's start with something very concrete

Suppose we want a function that prints a person's age.

We don't need generics:

```kotlin
fun printAge(age: Int) {
    println(age)
}
```

Fine.

But imagine we're writing a generic function that needs some particular capability.

For example, suppose we want to compare two values.

```kotlin
fun <T> bigger(a: T, b: T): T {
    // Which one is bigger?
}
```

We have a problem.

Kotlin doesn't know that `T` can be compared.

`T` could be:

```text
String
Int
User
Car
```

What does "bigger" mean for an arbitrary `User`?

There is no universal answer.

So we need to constrain `T`.

---

# The idea of an upper bound

Suppose we have:

```kotlin
fun <T : Number> process(value: T) {
    ...
}
```

Read it as:

> `T` can be any type, but it must be a subtype of `Number`.

So these are possibilities:

```text
Int
Long
Double
Float
Short
Byte
```

because they belong to the `Number` family.

But:

```text
String
Boolean
User
```

do not satisfy the bound.

So:

```kotlin
fun <T : Number>
```

is basically:

```text
T
↓
must be a Number
```

This is called an **upper bound**.

---

# Why is the constraint useful?

Because now Kotlin knows something about `T`.

Without a constraint:

```kotlin
fun <T> process(value: T) {
    // almost nothing can be assumed
}
```

With:

```kotlin
fun <T : Number> process(value: T) {
    // T belongs to Number's type family
}
```

we have more information.

We're narrowing the possibilities.

Think of it like this:

```text
<T>

could be almost anything

        ↓

<T : Number>

could be any Number subtype
```

Generics become much more useful once we can control what kinds of types are allowed.

---

# But there's something subtle about `Number`

You might immediately try:

```kotlin
fun <T : Number> doubleIt(value: T): T {
    return value * 2
}
```

and discover that this isn't as straightforward as you might expect.

Why?

Because:

> Knowing that something is a `Number` does not mean Kotlin can magically perform every numeric operation while preserving the exact type `T`.

For example, `Int * 2` gives an `Int`, `Double * 2` gives a `Double`, and so on.

The generic function doesn't automatically know what exact numeric subtype `T` is.

This is a valuable lesson:

> **A generic bound tells you what is guaranteed, not what you wish were guaranteed.**

`T : Number` guarantees:

> "This is some Number."

It does not guarantee:

> "I can treat this as an `Int`."

---

# A better constraint example

Interfaces make generic constraints easier to understand.

Suppose:

```kotlin
interface Printable {
    fun print()
}
```

Now we want a generic function that only accepts printable things.

We can write:

```kotlin
fun <T : Printable> printAnything(value: T) {
    value.print()
}
```

Now Kotlin knows that `T` has:

```kotlin
print()
```

because every allowed `T` must implement `Printable`.

For example:

```kotlin
class User : Printable {
    override fun print() {
        println("User")
    }
}

class Product : Printable {
    override fun print() {
        println("Product")
    }
}
```

Both work:

```kotlin
printAnything(User())
printAnything(Product())
```

But:

```kotlin
class Car
```

doesn't satisfy the constraint.

So:

```kotlin
printAnything(Car())
```

is rejected.

---

# This is where generic constraints really click

Without the bound:

```kotlin
fun <T> printAnything(value: T)
```

you can't safely write:

```kotlin
value.print()
```

because Kotlin knows nothing about `T`.

With:

```kotlin
fun <T : Printable> printAnything(value: T)
```

you've told Kotlin:

> Every legal `T` has the `Printable` contract.

So the compiler lets you use:

```kotlin
value.print()
```

That's the relationship I want you to understand.

---

# Generic constraint = permission to assume something

This is probably the simplest way to remember it.

When you write:

```kotlin
<T : Printable>
```

you're telling the compiler:

> "You may assume `T` has everything guaranteed by `Printable`."

When you write:

```kotlin
<T : Number>
```

you're telling it:

> "You may assume `T` is a subtype of `Number`."

So a generic constraint isn't merely restricting callers.

It's also **giving the implementation knowledge**.

That's why both sides benefit:

```text
constraint
├── rejects invalid types
└── gives the function more information about T
```

---

# Let's use a class instead of an interface

Suppose:

```kotlin
open class Animal {
    fun breathe() {
        println("Breathing")
    }
}
```

We can write:

```kotlin
fun <T : Animal> makeBreathe(value: T) {
    value.breathe()
}
```

Now `T` must be an `Animal` or subclass of `Animal`.

For example:

```kotlin
class Dog : Animal()
class Cat : Animal()
```

Both can be used.

But:

```kotlin
class Car
```

can't.

Again:

```text
T : Animal
```

means:

> "T belongs to the Animal family."

---

# You don't always need `<T>` at all

This is worth noticing.

Consider:

```kotlin
fun printAnimal(animal: Animal) {
    animal.breathe()
}
```

Why not just do that?

You absolutely can.

And often you should.

Generics become useful when the generic type itself needs to be preserved or used in a relationship.

For example:

```kotlin
fun <T : Animal> identity(animal: T): T {
    return animal
}
```

Now the exact subtype is preserved.

If:

```kotlin
val dog = Dog()

val result = identity(dog)
```

then `result` can remain known as a `Dog`, rather than merely being viewed as an `Animal`.

That's an important distinction.

---

# `Animal` vs `<T : Animal>`

Compare these.

### Version 1

```kotlin
fun identity(animal: Animal): Animal {
    return animal
}
```

The function says:

> "Give me an Animal, and I'll return an Animal."

### Version 2

```kotlin
fun <T : Animal> identity(animal: T): T {
    return animal
}
```

The function says:

> "Give me some specific subtype of Animal, and I'll return that same specific subtype."

So:

```text
Animal parameter
→ exact subtype information may be lost

T : Animal
→ relationship is preserved
```

This is one of the main reasons generics exist.

---

# Watch this carefully

Suppose:

```kotlin
class Dog : Animal() {
    fun bark() {
        println("Woof")
    }
}
```

Now:

```kotlin
val dog = Dog()
val result = identity(dog)
```

With:

```kotlin
fun identity(animal: Animal): Animal
```

`result` is treated as:

```text
Animal
```

So:

```kotlin
result.bark()
```

won't be available directly.

But with:

```kotlin
fun <T : Animal> identity(animal: T): T
```

the compiler can preserve:

```text
T = Dog
```

so `result` remains a `Dog`.

That's a very important practical reason for generic constraints.

---

# Constraints can work with interfaces

This is especially common.

Imagine:

```kotlin
interface HasId {
    val id: Int
}
```

Then:

```kotlin
fun <T : HasId> findId(item: T): Int {
    return item.id
}
```

Now `T` can be:

```kotlin
class User(
    override val id: Int
) : HasId

class Product(
    override val id: Int
) : HasId
```

Both work.

The function doesn't care whether `T` is a `User` or `Product`.

It only cares:

> "Does it have the capability required by `HasId`?"

That is a very powerful pattern.

---

# This gives us a better idea of what generics are for

Remember the generic function from the previous lesson:

```kotlin
fun <T> first(list: List<T>): T
```

That function didn't care what `T` was.

It only needed:

```text
a List<T>
```

Now imagine an operation that requires more knowledge.

For example:

```kotlin
interface Identifiable {
    val id: String
}
```

We can write:

```kotlin
fun <T : Identifiable> printId(value: T) {
    println(value.id)
}
```

Now we have:

```text
generic
+
constraint
=
generic code with guaranteed capabilities
```

That's the important idea.

---

# Multiple constraints

Now we can make things more interesting.

Suppose a type must satisfy **two conditions**.

Imagine:

```kotlin
interface HasId {
    val id: String
}

interface Printable {
    fun print()
}
```

We want:

> `T` must have an ID and be printable.

A type can't simply use:

```kotlin
<T : HasId, Printable>
```

in that position.

Instead, Kotlin gives us a `where` clause:

```kotlin
fun <T> process(value: T)
    where T : HasId,
          T : Printable {
    println(value.id)
    value.print()
}
```

Read this as:

> `T` must satisfy both `HasId` and `Printable`.

So:

```text
T
├── HasId
└── Printable
```

Both requirements must be true.

---

# Why can't we just put both in one `:`?

Because the first bound is special in Kotlin's syntax.

For example:

```kotlin
<T : Animal>
```

gives the primary upper bound.

If you have additional constraints, you can use:

```kotlin
where T : SomeInterface,
      T : AnotherInterface
```

The `where` clause becomes a clean way to express multiple requirements.

---

# Here's a realistic use

Suppose we have:

```kotlin
interface Entity {
    val id: String
}

interface Auditable {
    fun audit()
}
```

Then:

```kotlin
fun <T> process(entity: T)
    where T : Entity,
          T : Auditable {
    println("Processing ${entity.id}")
    entity.audit()
}
```

Now the function can safely assume:

```text
id exists
+
audit() exists
```

without knowing the exact concrete class.

That's generic programming with constraints.

---

# Multiple bounds aren't limited to two

You can express more constraints when the model requires them.

But don't get carried away.

If you find yourself writing:

```kotlin
where T : A,
      T : B,
      T : C,
      T : D,
      T : E,
      T : F
```

you should stop and ask:

> "Why is this generic function depending on six different abstractions?"

The language allows it.

That doesn't mean the design is good.

Again, we're learning a recurring programming lesson:

> **Capability doesn't determine good design.**

---

# Nullable types make generic bounds more interesting

Now let's revisit something you already know.

Suppose:

```kotlin
fun <T> test(value: T)
```

What can `T` be?

Potentially a nullable type.

For example:

```kotlin
String?
```

is possible as a type argument.

This is one reason the distinction between:

```text
Any
```

and:

```text
Any?
```

matters.

Remember from our earlier type-system lesson:

```text
Any
→ all non-null Kotlin values

Any?
→ nullable or non-null Kotlin values
```

So a generic constraint involving `Any` can have consequences for whether `T` is allowed to be nullable.

---

# Consider this

```kotlin
fun <T : Any> process(value: T)
```

This means:

> `T` must be a non-nullable type.

Why?

Because `T` has an upper bound of `Any`, and `Any` excludes `null`.

Compare:

```kotlin
fun <T> process(value: T)
```

which is more permissive.

This is an important pattern you'll eventually see in real Kotlin APIs:

```kotlin
<T : Any>
```

often means:

> "I'm deliberately saying this generic type must not be nullable."

Don't memorize it as a strange incantation.

Just connect it to the type hierarchy you already learned:

```text
Any
→ non-null types

Any?
→ nullable + non-null
```

---

# Why would a library care whether `T` is nullable?

Imagine a generic class that uses:

```text
T
```

as something that must always exist.

Allowing:

```text
T = String?
```

might make the API much harder to reason about.

So the library can say:

```kotlin
<T : Any>
```

and now:

```text
T cannot be nullable
```

The type system becomes part of the API contract.

---

# Generic constraints are really contracts

This is a useful way to think about them.

If I write:

```kotlin
fun <T : Printable> printAnything(value: T)
```

I'm not merely saying:

> "Some types aren't allowed."

I'm saying:

> "This function requires the capability represented by `Printable`."

And:

```kotlin
fun <T : Animal> feed(value: T)
```

says:

> "This function operates on the Animal family."

And:

```kotlin
fun <T : Any> store(value: T)
```

says:

> "This operation expects a non-null type."

So generic constraints are part of the function's **contract**.

---

# Let's connect this back to interfaces

You've already learned that an interface represents a contract.

For example:

```kotlin
interface Printable {
    fun print()
}
```

Now:

```kotlin
fun <T : Printable>
```

means:

> "The generic type must satisfy that contract."

So generic constraints and interfaces work together beautifully:

```text
interface
→ describes capability

generic constraint
→ says which capabilities this generic code requires
```

That's a very important relationship.

---

# Now here's something subtle: one bound can be a class

Suppose:

```kotlin
open class Animal
```

and:

```kotlin
fun <T : Animal> feed(animal: T)
```

That's fine.

But because a type can have only one superclass, you can't have:

```text
T : Animal
T : Vehicle
```

if both `Animal` and `Vehicle` are classes.

That would require the same type to have two class superclasses, which Kotlin's class inheritance model doesn't allow.

Multiple bounds work especially naturally when the extra bounds are interfaces.

Think:

```text
one superclass
+
multiple interfaces
```

That's the inheritance model you already learned.

Generic constraints follow that same reality.

---

# Let's look at a generic sorting example

Suppose we want:

```kotlin
fun <T> max(a: T, b: T): T
```

We can't compare arbitrary `T`.

But suppose `T` implements:

```kotlin
Comparable<T>
```

Now we can say:

```kotlin
fun <T : Comparable<T>> max(a: T, b: T): T {
    return if (a >= b) a else b
}
```

The exact operators and generic details are something we'll examine more carefully later, but conceptually:

```text
T
must know how to compare itself with T
```

So:

```text
generic
+
capability constraint
```

lets us perform an operation that isn't valid for arbitrary types.

This pattern is all over Kotlin and the standard library.

---

# Read `Comparable<T>` slowly

This:

```kotlin
Comparable<T>
```

is itself a generic type.

So now we have a type parameter inside a type parameter:

```text
T
↓
must implement Comparable<T>
```

That means:

> `T` must be comparable to another `T`.

For something like:

```text
Int
String
```

that makes intuitive sense.

You're getting a glimpse of why generics eventually become a deep topic.

The language is allowing us to express relationships **between types**, not merely name them.

---

# Here's another very important difference

Compare:

```kotlin
fun <T> doSomething(value: T)
```

with:

```kotlin
fun <T : Printable> doSomething(value: T)
```

The first says:

> "I can accept anything."

The second says:

> "I can accept many things, but only things satisfying `Printable`."

That means the second function has **less freedom at the boundary**, but **more knowledge inside the implementation**.

That's a recurring tradeoff in type design.

---

# Constraints can actually improve reuse

This sounds paradoxical.

You might think:

> "If I constrain `T`, I'm making my generic function less reusable."

Technically, yes, the set of accepted types becomes smaller.

But you're also making the function usable for a **whole family of types** that share a meaningful capability.

For example:

```kotlin
fun <T : Printable> printAnything(value: T)
```

can work with:

```text
User
Product
Invoice
Report
Receipt
...
```

as long as they implement `Printable`.

So the function doesn't need to know six concrete classes.

It works with an abstraction.

That's excellent reuse.

---

# And now let's look at a common mistake

Someone might write:

```kotlin
fun <T> printAnything(value: T) {
    if (value is Printable) {
        value.print()
    }
}
```

This is valid.

But ask yourself:

> Is the function supposed to work with literally anything, or does it fundamentally require a `Printable`?

If it fundamentally requires `Printable`, then:

```kotlin
fun <T : Printable> printAnything(value: T)
```

is more honest.

Or perhaps you don't even need generics:

```kotlin
fun printAnything(value: Printable)
```

The correct design depends on whether preserving the exact type `T` matters.

This is a really important decision.

---

# When generics are unnecessary

Suppose:

```kotlin
interface Printable {
    fun print()
}
```

and all we want is:

```kotlin
fun printThing(value: Printable) {
    value.print()
}
```

That's probably enough.

We don't automatically gain anything by writing:

```kotlin
fun <T : Printable> printThing(value: T) {
    value.print()
}
```

unless we need something about the exact `T`.

This is another rule I want you to absorb:

> **Don't use generics just because the function can be written generically.**

Use them when preserving or relating types actually matters.

---

# A great example of when `T` DOES matter

Suppose:

```kotlin
fun <T : Animal> identity(value: T): T {
    return value
}
```

Now the exact subtype matters.

If I pass:

```text
Dog
```

I want:

```text
Dog
```

back.

That's a meaningful generic relationship.

But if I only want:

```text
call breathe()
```

then:

```kotlin
fun breathe(animal: Animal)
```

might be simpler.

---

# Here's the pattern emerging

We're gradually moving through levels of specificity.

### Completely generic

```kotlin
<T>
```

Meaning:

> I know almost nothing about `T`.

### Constrained generic

```kotlin
<T : Animal>
```

Meaning:

> I know `T` belongs to the Animal family.

### Generic with multiple constraints

```kotlin
<T>
where T : Entity,
      T : Printable
```

Meaning:

> I know `T` satisfies several contracts.

So generic constraints are basically about increasing what your code is allowed to **know** about a type without giving up generic reuse.

---

# One last example: a reusable function

Let's build something where the constraint is actually necessary.

Suppose:

```kotlin
interface HasLength {
    val length: Int
}
```

Then:

```kotlin
class Word(
    val text: String
) : HasLength {
    override val length: Int
        get() = text.length
}
```

Now:

```kotlin
fun <T : HasLength> isLong(value: T): Boolean {
    return value.length > 10
}
```

This function doesn't care what `T` actually is.

It only cares that:

```text
T has a length
```

That is exactly what generic constraints are good at.

---

# Your turn

I want you to answer these conceptually before you code.

### 1

What does this mean?

```kotlin
fun <T : Animal> feed(animal: T)
```

Don't say merely:

> "T is generic."

Explain what the `: Animal` gives us.

---

### 2

Why can't we safely write this?

```kotlin
fun <T> printLength(value: T) {
    println(value.length)
}
```

What does Kotlin know about arbitrary `T`?

---

### 3

Why might this:

```kotlin
fun print(value: Printable)
```

be better than:

```kotlin
fun <T : Printable> print(value: T)
```

in some cases?

And when would the generic version actually be useful?

---

### 4

What does this mean?

```kotlin
fun <T> process(value: T)
    where T : HasId,
          T : Printable
```

What must be true about `T`?

---

# Coding exercise

Create:

```kotlin
interface HasName {
    val name: String
}
```

Then create two classes:

```kotlin
User
Product
```

that both implement `HasName`.

Now write:

```kotlin
fun <T : HasName> printName(value: T)
```

and use it with both types.

Then deliberately try passing a class that does **not** implement `HasName`.

Look at the compiler error.

That's the simplest possible demonstration of:

```text
generic
+
constraint
+
compiler-enforced contract
```

---

# One challenge

Write:

```kotlin
interface HasId {
    val id: Int
}

interface Describable {
    fun describe(): String
}
```

Then create a generic function that only accepts types satisfying **both** contracts.

Inside the function, it should be able to use:

```text
id
describe()
```

without casts.

The trick is the `where` clause.

---

## Where we are now

You've gone from:

```text
T = "some type"
```

to:

```text
T = some type satisfying a contract
```

That's a big conceptual step.

And notice how this connects almost everything we've learned:

```text
interfaces
    ↓
contracts/capabilities

generics
    ↓
abstract over types

constraints
    ↓
tell Kotlin which capabilities T guarantees

compiler
    ↓
lets your generic code safely use those capabilities
```

Next, we're going to get into one of the **most important and most confusing parts of Kotlin generics: variance** — `in` and `out`.

That's where you'll encounter a question that seems innocent but is surprisingly deep:

> **If `Dog` is an `Animal`, why isn't `List<Dog>` automatically a `List<Animal>`?**

Once you understand that, a huge amount of Kotlin's generic type system starts making sense.