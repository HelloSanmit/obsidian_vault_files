
# Phase 8 — Lesson 31: Variance — Understanding `out` and `in`

This is one of those topics where the syntax is tiny but the idea underneath is genuinely important.

You already know this:

```kotlin
class Box<T>
```

and:

```kotlin
List<T>
```

Today we need to answer a question that looks simple:

> If a `Dog` is an `Animal`, why isn't a `List<Dog>` automatically a `List<Animal>`?

At first, that feels like it should obviously work.

Let's see why it sometimes **should** work, and sometimes absolutely **must not**.

---

## Start with ordinary inheritance

Suppose:

```kotlin
open class Animal

class Dog : Animal()
class Cat : Animal()
```

Obviously:

```kotlin
val dog = Dog()
```

can be treated as:

```kotlin
val animal: Animal = dog
```

because a `Dog` is an `Animal`.

That's ordinary subtype polymorphism.

So we have:

```text
Dog
 ↓
Animal
```

Now let's put those types inside another type.

Suppose:

```kotlin
List<Dog>
```

Should that be usable as:

```kotlin
List<Animal>
```

?

The answer is:

**For Kotlin's read-only `List`, yes.**

And that is not an accident.

But let's first understand why it would be dangerous for a mutable collection.

---

# Imagine a mutable list

Suppose Kotlin allowed this:

```kotlin
val dogs: MutableList<Dog> = mutableListOf()

val animals: MutableList<Animal> = dogs
```

That looks harmless.

`Dog` is an `Animal`.

So why not?

Because now I can do:

```kotlin
animals.add(Cat())
```

And what have I just done?

I've inserted a `Cat` into:

```text
dogs
```

But `dogs` is supposed to contain only `Dog`.

Now you've created a contradiction.

The same actual list is being viewed through two different type descriptions:

```text
dogs
→ "only Dogs"

animals
→ "any Animals"
```

And the second view allows something the first view cannot safely contain.

That's the core problem.

---

# This is the reason `MutableList<Dog>` is not a `MutableList<Animal>`

Kotlin therefore treats:

```kotlin
MutableList<Dog>
```

and:

```kotlin
MutableList<Animal>
```

as unrelated generic types for assignment purposes.

This is called **invariance**.

So:

```text
Dog <: Animal
```

does **not** imply:

```text
MutableList<Dog> <: MutableList<Animal>
```

Why?

Because a mutable list does two things:

```text
you can READ from it
+
you can WRITE to it
```

And those two directions create conflicting requirements.

This is the doorway into variance.

---

# Now look at a read-only list

Suppose:

```kotlin
val dogs: List<Dog> = listOf(Dog())
```

and:

```kotlin
val animals: List<Animal> = dogs
```

This is safe.

Why?

Because through `animals`, you can read:

```kotlin
val animal = animals[0]
```

and yes, every `Dog` is an `Animal`.

But you cannot do:

```kotlin
animals.add(Cat())
```

because `List` is read-only.

So nobody can use the broader type to inject a `Cat` into the original list.

That's why Kotlin can safely allow:

```text
List<Dog>
      ↓
List<Animal>
```

This is **covariance**.

---

# The word "variance" is just describing this relationship

These three ideas are the ones you need:

### Invariant

```text
A <: B

but

Container<A> and Container<B>
are not related
```

### Covariant

```text
A <: B

then

Container<A> <: Container<B>
```

### Contravariant

The relationship goes the other way:

```text
A <: B

then

Container<B> <: Container<A>
```

At first, contravariance looks bizarre.

We'll build it carefully.

---

# Let's understand covariance first

Suppose:

```kotlin
open class Animal

class Dog : Animal()
class Cat : Animal()
```

Kotlin's `List` behaves conceptually like:

```kotlin
interface List<out T>
```

That `out` is saying that `List` is **covariant** in `T`.

So if:

```text
Dog is an Animal
```

then:

```text
List<Dog> can be used where List<Animal> is expected
```

That makes intuitive sense if you think:

> "I'm only taking values out."

A list of dogs is certainly a list of animals.

Every element you take from it is an `Animal`.

---

# Think of `List<Dog>` as a producer

This is the most useful intuition for `out`.

A producer gives you values.

A:

```kotlin
List<Dog>
```

can give you:

```text
Dog
```

And since a `Dog` is an `Animal`, it can also give you something that can be treated as:

```text
Animal
```

So:

```text
producer of Dog
→ can act as
producer of Animal
```

That's covariance.

And that's where the slogan comes from:

> **`out` means producer.**

Not as a perfect mathematical definition, but as a very useful programming intuition.

---

# Let's make a generic producer ourselves

Imagine:

```kotlin
interface Source<T> {
    fun get(): T
}
```

A source produces a `T`.

Suppose we write:

```kotlin
interface Source<out T> {
    fun get(): T
}
```

Now:

```kotlin
class DogSource : Source<Dog> {
    override fun get(): Dog {
        return Dog()
    }
}
```

Because `Source` is covariant:

```kotlin
val source: Source<Animal> = DogSource()
```

is allowed.

Why?

Because anything `DogSource` gives us is a `Dog`, and every `Dog` is an `Animal`.

So the broader consumer can safely receive it.

---

# But `out` has a rule

If a type parameter is declared as:

```kotlin
out T
```

then the generic type cannot freely use `T` as an input.

Why?

Because if the thing is supposed to be a producer of `T`, letting you feed arbitrary `T`s into it would break the safety guarantee.

For example, this kind of interface would not make sense:

```kotlin
interface Source<out T> {
    fun get(): T
    fun put(value: T)
}
```

The compiler rejects this use of `T`.

Why?

Because you've declared:

> "This type is covariant in T."

But then you're trying to consume `T` as input too.

That is exactly what covariance is designed to prevent.

---

# Think "out" as a one-way door

For a covariant type:

```kotlin
out T
```

the type parameter flows mostly:

```text
inside
  ↓
outside
```

The object can give you a `T`.

That's safe.

But it should not accept arbitrary `T` values from you.

So:

```text
out
→ T comes OUT
```

That's why the keyword is named `out`.

---

# Now let's turn the problem around

Suppose you don't have something that produces values.

Suppose you have something that **consumes** them.

For example:

```kotlin
interface Sink<T> {
    fun consume(value: T)
}
```

A sink takes values.

A good example is a:

```kotlin
Comparator<T>
```

or:

```kotlin
Printer<T>
```

or:

```kotlin
Handler<T>
```

These things tend to consume a value rather than produce one.

Now covariance isn't the right relationship.

We need the opposite.

---

# Let's use a handler

Suppose:

```kotlin
open class Animal

class Dog : Animal()
class Cat : Animal()
```

And:

```kotlin
interface Handler<T> {
    fun handle(value: T)
}
```

Now imagine we have:

```kotlin
class AnimalHandler : Handler<Animal> {
    override fun handle(value: Animal) {
        println("Handled animal")
    }
}
```

Can an `AnimalHandler` handle a `Dog`?

Absolutely.

A `Dog` is an `Animal`.

So:

```text
Handler<Animal>
```

can safely handle:

```text
Dog
```

Now imagine a place in the program that says:

```kotlin
val dogHandler: Handler<Dog>
```

Could we give it:

```kotlin
AnimalHandler
```

?

Yes, conceptually.

Because if you promise:

> "I can handle every Dog"

an object that can handle:

> "every Animal"

certainly satisfies that promise.

This is the opposite direction from covariance.

---

# That's contravariance

Kotlin expresses that with:

```kotlin
interface Handler<in T> {
    fun handle(value: T)
}
```

The `in` means:

> This type consumes `T`.

Now the subtype relationship reverses.

If:

```text
Dog <: Animal
```

then:

```text
Handler<Animal> <: Handler<Dog>
```

That looks backwards at first.

Let's make it intuitive.

---

# Think about what the requirement means

Suppose some function expects:

```kotlin
fun registerHandler(handler: Handler<Dog>)
```

What does that requirement mean?

It means:

> "Give me something capable of handling a Dog."

Now give it:

```kotlin
AnimalHandler
```

Is that okay?

Yes.

Because `AnimalHandler` can handle **any Animal**, which includes Dogs.

So:

```text
can handle Animal
→ certainly can handle Dog
```

That's why:

```text
Handler<Animal>
```

can stand in for:

```text
Handler<Dog>
```

This is contravariance.

---

# The easiest mental trick

For `out`:

> **I produce values.**

For `in`:

> **I consume values.**

So:

```text
out
→ give values to me

in
→ I give values to you
```

Or, even more simply:

```text
out → reading
in  → writing/consuming
```

That's not the entire formal theory, but it's an excellent working intuition.

---

# Why `MutableList<T>` is invariant

Now you can see the problem more clearly.

A mutable list does both:

```text
read T
write T
```

So it can't safely be declared:

```kotlin
MutableList<out T>
```

because then it couldn't accept arbitrary `T` values.

And it can't simply be:

```kotlin
MutableList<in T>
```

because we also need to read values back as `T`.

So it stays:

```kotlin
MutableList<T>
```

Invariant.

This is not arbitrary.

The operations available on the abstraction determine what variance is safe.

---

# This is the real rule behind variance

Instead of memorizing:

```text
List → covariant
MutableList → invariant
```

understand this:

### A type is covariant when it can safely expose `T` without accepting arbitrary `T`s.

### A type is contravariant when it can safely accept `T` without promising to produce `T`.

### A type that both consumes and produces `T` generally has to be invariant.

That is the conceptual heart of variance.

---

# Let's look at `Comparable`

You've probably seen:

```kotlin
Comparable<T>
```

Kotlin defines this concept in a contravariant way:

```text
Comparable<in T>
```

Why does that make sense?

A comparable object is used for comparison.

Conceptually:

```kotlin
interface Comparable<in T> {
    operator fun compareTo(other: T): Int
}
```

It's **consuming** another `T`.

So `in` makes sense.

A type that can compare itself against a broad family of values may also be usable where a narrower comparison is expected, subject to the actual type relationships involved.

You don't need to master that API today.

I want you to recognize why `in` appears there.

---

# Function types are a fantastic example

Here's where variance becomes really intuitive.

Suppose:

```kotlin
val f: (Animal) -> String
```

This is a function that:

```text
takes Animal
returns String
```

Now imagine:

```kotlin
val dogFunction: (Dog) -> String
```

Which one is safer to use where a:

```kotlin
(Dog) -> String
```

is expected?

The function that accepts:

```text
Animal
```

or the function that accepts:

```text
Dog
```

Think carefully.

If I give you a function that accepts **any Animal**, can you safely use it where the caller promises to give you a Dog?

Yes.

A Dog is an Animal.

So:

```text
(Animal) -> String
```

can be used where:

```text
(Dog) -> String
```

is expected.

That's contravariance in action.

---

# But what about the return type?

Suppose:

```text
() -> Dog
```

versus:

```text
() -> Animal
```

A function that returns a `Dog` can safely be used where a function returning an `Animal` is expected.

Why?

Because:

```text
Dog is an Animal
```

So:

```text
() -> Dog
```

can act as:

```text
() -> Animal
```

That's covariance in the **return position**.

So function types are interesting because:

```text
parameter types → contravariant
return type → covariant
```

That's a deeper but very important pattern.

You don't have to memorize the phrase yet.

Just notice:

```text
things coming OUT
→ covariance

things going IN
→ contravariance
```

---

# Let's visualize function variance

Consider:

```kotlin
(Animal) -> Dog
```

There are two type positions:

```text
(Animal) → Dog
   ↑        ↑
  input    output
```

The input uses contravariant behavior.

The output uses covariant behavior.

So:

```text
input
→ in

output
→ out
```

This relationship is one of the reasons function types in Kotlin work the way they do.

---

# This also explains some generic declarations

Suppose you see:

```kotlin
interface Producer<out T>
```

Immediately think:

> This thing produces `T`.

If you see:

```kotlin
interface Consumer<in T>
```

think:

> This thing consumes `T`.

That is much more useful than memorizing the keywords mechanically.

---

# Now let's connect it to our previous lessons

You learned interfaces:

```kotlin
interface Printer
```

You learned generics:

```kotlin
Box<T>
```

Now we're combining the two:

```kotlin
interface Source<out T>
interface Sink<in T>
```

This is where the type system gets interesting.

We're no longer saying just:

> "T is some type."

We're saying:

> "T has a particular directional relationship with this abstraction."

That direction determines which subtype substitutions are safe.

---

# A concrete example with producers and consumers

Suppose:

```kotlin
open class Animal {
    val name = "animal"
}

class Dog : Animal() {
    fun bark() = println("Woof")
}
```

A producer:

```kotlin
interface Producer<out T> {
    fun produce(): T
}
```

Then:

```kotlin
class DogProducer : Producer<Dog> {
    override fun produce(): Dog {
        return Dog()
    }
}
```

We can write:

```kotlin
val producer: Producer<Animal> = DogProducer()
```

because a producer of Dogs is safe to treat as a producer of Animals.

Every Dog produced is an Animal.

---

Now a consumer:

```kotlin
interface Consumer<in T> {
    fun consume(value: T)
}
```

And:

```kotlin
class AnimalConsumer : Consumer<Animal> {
    override fun consume(value: Animal) {
        println(value.name)
    }
}
```

Then:

```kotlin
val consumer: Consumer<Dog> = AnimalConsumer()
```

is safe.

Why?

Because anything that can consume an Animal can definitely consume a Dog.

That's the symmetry:

```text
Producer<Dog>
→ Producer<Animal>

Consumer<Animal>
→ Consumer<Dog>
```

The arrows go in opposite directions.

---

# This is where the names finally make sense

### `out`

```text
Producer<out T>
```

The relationship moves outward:

```text
Dog
→ Animal

Producer<Dog>
→ Producer<Animal>
```

### `in`

```text
Consumer<in T>
```

The relationship moves inward in the opposite direction:

```text
Dog
← Animal

Consumer<Dog>
← Consumer<Animal>
```

That's why the keywords are named `in` and `out`.

---

# Don't confuse variance with inheritance

This is another important distinction.

Suppose:

```kotlin
class Dog : Animal()
```

That's a relationship between the types themselves:

```text
Dog <: Animal
```

Variance describes what happens when those types are inserted into a **generic type**:

```text
Dog
↓
Animal

what about:
Container<Dog>
Container<Animal>
?
```

So:

```text
inheritance
→ relationship between types

variance
→ relationship between parameterized types based on those type relationships
```

That's a subtle but essential difference.

---

# Why Kotlin's `List` can be covariant

Now when you see:

```kotlin
val animals: List<Animal> = dogs
```

you should no longer think:

> "Kotlin just decided to allow this."

You should think:

```text
List is a producer/read-only abstraction
→ it can safely expose Dog as Animal
→ therefore it can be covariant
```

And internally, Kotlin's `List` type is designed with covariance.

---

# Why `MutableList` can't do the same

Because:

```text
MutableList
├── reads T
└── writes T
```

If it were covariant, you could create the exact unsafe situation we started with:

```text
MutableList<Dog>
→ treated as MutableList<Animal>
→ insert Cat
→ original Dog list becomes invalid
```

So the type system refuses the substitution.

This is a perfect example of the type system preventing a logical bug **before runtime**.

---

# There's an elegant general principle here

Whenever you're looking at a generic abstraction, ask:

> Does it mostly produce values, consume values, or do both?

### Mostly produces

Think:

```text
out
covariant
```

### Mostly consumes

Think:

```text
in
contravariant
```

### Both

Think:

```text
invariant
```

This is probably the most useful practical framework for variance.

---

# A warning: "producer = only returns T" is a simplification

In real Kotlin, the rules about where a type parameter may appear are more precise than simply looking at whether the class "feels like a producer."

For example, variance interacts with:

- generic properties
    
- function parameter types
    
- return types
    
- nested generic types
    
- private members
    
- overriding
    
- type projections
    

So don't use the slogan as the formal definition.

Use it as your first intuition.

As we go deeper, we'll sharpen it.

---

# One more thing: variance is about type safety, not runtime behavior

When you write:

```kotlin
List<Dog>
```

versus:

```kotlin
List<Animal>
```

variance isn't changing how the list physically stores the elements.

It changes **which type relationships the compiler permits**.

This is primarily a type-system issue.

That's why you can think:

```text
variance
→ compile-time reasoning
```

rather than:

```text
variance
→ some special runtime conversion
```

The actual runtime representation is a separate question.

---

# Let's tie it back to `Any`

Suppose:

```kotlin
val names: List<String> = listOf("A", "B")
```

Since `String` is a subtype of `Any`, covariance means:

```kotlin
val things: List<Any> = names
```

is valid.

And that makes sense:

```text
List<String>
```

is a read-only producer of Strings.

Every String is an Any.

So it's safe to view the list as:

```text
List<Any>
```

because you're only promising:

> "Everything I give you is some `Any`."

And that's true.

---

# But the reverse doesn't work

This does not work:

```kotlin
val things: List<Any> = listOf("A", 10, true)
val names: List<String> = things
```

Why?

Because `things` can contain:

```text
String
Int
Boolean
```

and not every `Any` is a `String`.

So:

```text
List<Any>
→ List<String>
```

would be unsafe.

Covariance goes only in the subtype direction:

```text
String
→ Any

List<String>
→ List<Any>
```

not backward.

---

# Here's the picture I want in your head

```text
          Ordinary inheritance

          Dog
           │
           ▼
         Animal


          Covariance

      List<Dog>
           │
           ▼
      List<Animal>


       Contravariance

    Consumer<Animal>
           │
           ▼
      Consumer<Dog>


       Invariance

   MutableList<Dog>

          X

   MutableList<Animal>
```

Don't memorize the arrows independently.

Understand **why** they're there.

---

# A very important question

Suppose I tell you:

```kotlin
interface Producer<out T>
```

and:

```kotlin
interface Consumer<in T>
```

What would you expect each interface to be allowed to do with `T`?

You should be able to answer before seeing the methods.

```text
Producer<out T>
→ should mainly give T to you

Consumer<in T>
→ should mainly receive T from you
```

That's the point at which variance starts becoming intuitive.

---

# Your exercises

Don't try to memorize the definitions yet.

## Exercise 1

We have:

```kotlin
open class Animal
class Dog : Animal()
```

Which of these should be safe?

```kotlin
val a: List<Animal> = listOf<Dog>(Dog())
```

```kotlin
val b: MutableList<Animal> = mutableListOf<Dog>(Dog())
```

Explain why one can work and the other cannot.

---

## Exercise 2

Imagine:

```kotlin
interface Producer<out T> {
    fun produce(): T
}
```

Why should this be safe?

```kotlin
val producer: Producer<Animal> = DogProducer()
```

where `DogProducer` implements `Producer<Dog>`?

---

## Exercise 3

Imagine:

```kotlin
interface Consumer<in T> {
    fun consume(value: T)
}
```

Why should this be safe?

```kotlin
val consumer: Consumer<Dog> = AnimalConsumer()
```

where `AnimalConsumer` implements `Consumer<Animal>`?

The direction is deliberately opposite to the producer example.

---

## Exercise 4

Explain this in plain English:

```kotlin
List<Dog> → List<Animal>
```

Why is it safe?

And explain why:

```kotlin
MutableList<Dog> → MutableList<Animal>
```

would be dangerous.

---

# One coding experiment I'd really like you to do

Write:

```kotlin
open class Animal {
    open fun speak() {
        println("Animal sound")
    }
}

class Dog : Animal() {
    override fun speak() {
        println("Woof")
    }
}
```

Then create:

```kotlin
val dogs: List<Dog> = listOf(Dog())
```

and:

```kotlin
val animals: List<Animal> = dogs
```

Now try to reason:

> What operations can I perform through `animals` that are still guaranteed to be safe?

You'll realize the answer is essentially:

> I can read animals, because every dog is an animal.

Now imagine the same experiment with `MutableList`.

Ask:

> Could someone use the broader `MutableList<Animal>` view to insert something that isn't a `Dog`?

That one thought experiment explains a huge amount of variance.

---

# And one final connection

You already learned that Kotlin tries to make invalid states harder to express.

Variance is another instance of that philosophy.

The compiler is protecting you from saying:

```text
"Here's a list containing only Dogs"
```

and then treating it as:

```text
"Here's a list where I am allowed to insert any Animal."
```

Those two promises are incompatible.

So Kotlin's type system says:

> No. Those types aren't interchangeable.

But it happily allows:

```text
"Here's a read-only producer of Dogs"
```

to become:

```text
"Here's a read-only producer of Animals."
```

because that promise is safe.

That is the heart of variance.

---

## Where we go next

We've learned the idea behind:

```text
invariant
covariant
contravariant
out
in
```

But there are still two important pieces before I'd consider generics/variance properly understood:

**type projections** — especially `in`/`out` when you use an existing generic type rather than declaring one — and **star projections**, the `*` you sometimes see in Kotlin code.

Those are where you'll finally understand things like:

```kotlin
List<*>
MutableList<out Animal>
MutableList<in Dog>
```

without treating them as mysterious syntax.

That is the next part.