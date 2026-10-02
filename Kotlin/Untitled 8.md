
# Phase 8 — Lesson 29: Generics — Why Do We Need Them?

Now we're entering a different part of Kotlin.

So far, most of our code has been very specific.

For example:

```kotlin
fun double(number: Int): Int {
    return number * 2
}
```

This works beautifully for `Int`.

But what if I want the same idea for `Double`?

I'd need:

```kotlin
fun double(number: Double): Double {
    return number * 2
}
```

And what about `Long`?

```kotlin
fun double(number: Long): Long {
    return number * 2
}
```

You can already see where this goes.

We're writing essentially the same idea again and again, only because the **type** is different.

Generics solve a much broader version of this problem.

---

# First, forget the word "generic"

Suppose I want a box.

A very stupid box, just to illustrate the idea:

```kotlin
class IntBox(
    val value: Int
)
```

This gives me:

```kotlin
val box = IntBox(10)

println(box.value)
```

Fine.

Then I want to store a string:

```kotlin
class StringBox(
    val value: String
)
```

Then a Boolean:

```kotlin
class BooleanBox(
    val value: Boolean
)
```

Then a `User`:

```kotlin
class UserBox(
    val value: User
)
```

Now I have a family of almost identical classes:

```text
IntBox
StringBox
BooleanBox
UserBox
...
```

And the only thing changing is the type of `value`.

That is a sign that we're missing something.

---

# What if the box didn't care what the type was?

That's the idea:

```kotlin
class Box<T>(
    val value: T
)
```

That `<T>` is the key.

Now:

```kotlin
val a = Box(10)
val b = Box("Hello")
val c = Box(true)
```

Kotlin figures out:

```text
a → Box<Int>
b → Box<String>
c → Box<Boolean>
```

We've written **one class**.

But that class can work with many types while still remembering exactly what type it contains.

That's the central purpose of generics.

---

# What is `T`?

At this point, don't think:

> "`T` is some special Kotlin type."

It isn't.

`T` is a **type parameter**.

It's a placeholder.

When we write:

```kotlin
class Box<T>(
    val value: T
)
```

we're saying:

> "I don't know the type yet. Whoever uses this class will tell me."

Then:

```kotlin
Box(10)
```

effectively gives:

```text
T = Int
```

while:

```kotlin
Box("Hello")
```

gives:

```text
T = String
```

So:

```text
T
↓
placeholder for a type
```

That's the entire idea.

---

# Why not use `Any`?

This is where generics become really important.

You might think:

> Couldn't we just write this?

```kotlin
class Box(
    val value: Any
)
```

Yes.

But look at what happens.

```kotlin
val box = Box("Hello")
```

The property type is:

```text
Any
```

So later:

```kotlin
println(box.value.length)
```

doesn't work directly.

Why?

Because Kotlin only knows:

```text
box.value : Any
```

And not every `Any` has a `length`.

You'd have to inspect or cast it.

That throws away useful type information.

---

# Generics preserve the type

Now:

```kotlin
class Box<T>(
    val value: T
)
```

and:

```kotlin
val box = Box("Hello")
```

Kotlin knows:

```text
box : Box<String>
box.value : String
```

So:

```kotlin
println(box.value.length)
```

works.

That's one of the biggest reasons generics exist:

> **They let you write general code without losing information about the specific type being used.**

---

# This is a very important distinction

Compare:

```text
Any
↓
"I can hold anything, but I've forgotten what it specifically is."
```

with:

```text
T
↓
"I can hold anything, and I remember the specific type supplied here."
```

That's the difference.

---

# Let's make this more obvious

Suppose:

```kotlin
class Box<T>(
    val value: T
)
```

Now:

```kotlin
val numberBox = Box(42)
```

Kotlin knows:

```text
numberBox.value → Int
```

So:

```kotlin
println(numberBox.value + 10)
```

works.

Now:

```kotlin
val textBox = Box("Kotlin")
```

Kotlin knows:

```text
textBox.value → String
```

So:

```kotlin
println(textBox.value.length)
```

works.

Same class.

Different type.

And the compiler preserves the distinction.

---

# This is the pattern behind `List`

You've already been using generics for quite some time.

You wrote things like:

```kotlin
val names: List<String>
```

and:

```kotlin
val ages: List<Int>
```

Now you can understand what that actually means.

`List` is generic.

Think roughly like:

```kotlin
interface List<T>
```

where `T` represents:

> the type of elements in this list.

So:

```kotlin
List<String>
```

means:

> a list whose elements are `String`.

And:

```kotlin
List<Int>
```

means:

> a list whose elements are `Int`.

You weren't just learning collection syntax earlier.

You were already using generic types without yet understanding the machinery behind them.

---

# And `Map` is generic twice

Consider:

```kotlin
val ages: Map<String, Int>
```

That's two type parameters:

```text
Map<Key, Value>
```

So:

```kotlin
Map<String, Int>
```

means:

```text
key → String
value → Int
```

While:

```kotlin
Map<Int, String>
```

means:

```text
key → Int
value → String
```

Same generic class.

Different choices for its type parameters.

---

# So generics are already everywhere

You've already written:

```kotlin
List<String>
List<Int>
Set<String>
Map<String, Int>
```

and now you can finally understand what the angle brackets mean.

They aren't magic collection syntax.

They're specifying the type arguments used for a generic type.

---

# Let's make our own generic function

Suppose we want a function that takes something and returns exactly the same thing.

Without generics, what would we do?

Maybe:

```kotlin
fun identity(value: Any): Any {
    return value
}
```

But now:

```kotlin
val x = identity("Hello")
```

gives us an `Any`, not specifically a `String`.

We've lost information.

Instead:

```kotlin
fun <T> identity(value: T): T {
    return value
}
```

Now:

```kotlin
val x = identity("Hello")
```

gives:

```text
x : String
```

And:

```kotlin
val y = identity(42)
```

gives:

```text
y : Int
```

The function says:

> "Whatever type you give me, I'll give that same type back."

That's something `Any` can't express cleanly.

---

# Read this function carefully

```kotlin
fun <T> identity(value: T): T {
    return value
}
```

The pieces are:

```text
fun
↓
function declaration

<T>
↓
introduce a type parameter named T

value: T
↓
parameter has type T

: T
↓
return type is T
```

So the relationship is:

```text
input type
   =
output type
```

If the input is `String`, the output is `String`.

If the input is `Int`, the output is `Int`.

If the input is `User`, the output is `User`.

That relationship is the real power of the generic function.

---

# This is better than saying "anything"

Compare:

```kotlin
fun identity(value: Any): Any
```

with:

```kotlin
fun <T> identity(value: T): T
```

The first says:

> "I accept anything and return some `Any`."

The second says:

> "I accept a value of some type, and I promise to return that same type."

The second contains more information.

Generics are about expressing **relationships between types**.

That's a phrase I want you to remember.

---

# Here's another example

Suppose I have:

```kotlin
fun first(list: List<Any>): Any {
    return list[0]
}
```

That loses information.

If I pass:

```kotlin
val names = listOf("Alex", "Sam", "Priya")
```

the function returns `Any`.

But I know the answer is a `String`.

So instead:

```kotlin
fun <T> first(list: List<T>): T {
    return list[0]
}
```

Now:

```kotlin
val name = first(names)
```

gives:

```text
name : String
```

And:

```kotlin
val numbers = listOf(10, 20, 30)
val number = first(numbers)
```

gives:

```text
number : Int
```

One function handles both.

That's the power of generic abstraction.

---

# Notice something subtle

Our generic function isn't actually doing anything type-specific.

It doesn't care whether `T` is:

```text
String
Int
User
Order
Cat
List<Int>
```

It just preserves the type relationship.

That's when generics shine.

They are excellent when your algorithm doesn't care about the exact type, but the compiler still needs to know what that type is.

---

# Generic classes can have multiple type parameters

For example:

```kotlin
class PairBox<A, B>(
    val first: A,
    val second: B
)
```

Then:

```kotlin
val pair = PairBox("Alex", 25)
```

means approximately:

```text
A = String
B = Int
```

So:

```text
pair.first  → String
pair.second → Int
```

You could also have:

```kotlin
val pair = PairBox(true, 3.14)
```

which gives:

```text
A = Boolean
B = Double
```

The names `A` and `B` aren't magical.

They're just type-parameter names.

You could call them:

```kotlin
<X, Y>
```

or:

```kotlin
<First, Second>
```

although conventional short names are common for simple generic parameters.

---

# Naming conventions

You'll often see:

```text
T
K
V
E
R
```

These aren't special keywords.

They're conventions.

Common meanings include:

```text
T → Type
K → Key
V → Value
E → Element
R → Result
```

So:

```kotlin
Map<K, V>
```

usually means:

```text
Map<KeyType, ValueType>
```

while:

```kotlin
List<T>
```

means:

```text
List<ElementType>
```

The compiler doesn't care whether you use `T` or `Banana`.

Humans do.

Use meaningful names when the generic type has an important role.

---

# A very useful generic class

Let's build a little `Stack`.

A stack is conceptually:

```text
last in → first out
```

Like putting plates on top of each other.

We could hard-code it for `Int`:

```kotlin
class IntStack {
    private val items = mutableListOf<Int>()

    fun push(value: Int) {
        items.add(value)
    }

    fun pop(): Int {
        return items.removeAt(items.lastIndex)
    }
}
```

But why should a stack only work with integers?

We could make it generic:

```kotlin
class Stack<T> {

    private val items = mutableListOf<T>()

    fun push(value: T) {
        items.add(value)
    }

    fun pop(): T {
        return items.removeAt(items.lastIndex)
    }
}
```

Now:

```kotlin
val numbers = Stack<Int>()
numbers.push(10)
numbers.push(20)
```

and:

```kotlin
val names = Stack<String>()
names.push("Alex")
names.push("Sam")
```

use the same implementation.

That's a very natural use of generics.

---

# The generic type flows through the class

Look at:

```kotlin
class Stack<T> {

    private val items = mutableListOf<T>()

    fun push(value: T) {
        items.add(value)
    }

    fun pop(): T {
        return items.removeAt(items.lastIndex)
    }
}
```

Once the user chooses:

```text
Stack<Int>
```

then all those `T`s effectively become `Int`.

So mentally:

```text
T = Int
```

and the class behaves like:

```text
items → MutableList<Int>

push(value: Int)

pop(): Int
```

If:

```text
T = String
```

then:

```text
items → MutableList<String>

push(value: String)

pop(): String
```

This is why generics are sometimes described as **parameterized types**.

You're parameterizing a type by another type.

---

# Generic functions can be inside generic classes

And this is where the concepts start combining.

```kotlin
class Box<T>(
    val value: T
) {
    fun describe(): String {
        return "Value: $value"
    }
}
```

Here `T` belongs to the class.

So:

```kotlin
val box = Box(42)
```

creates a:

```text
Box<Int>
```

and its `value` is an `Int`.

---

# But functions can introduce their own type parameters

Consider:

```kotlin
class Box<T>(
    val value: T
) {
    fun <R> transform(transformer: (T) -> R): R {
        return transformer(value)
    }
}
```

Now we have **two type parameters** conceptually:

```text
T
→ belongs to Box

R
→ belongs to transform()
```

For example:

```kotlin
val box = Box(10)

val result = box.transform {
    it.toString()
}
```

Here:

```text
T = Int
R = String
```

So:

```text
Box<Int>
        ↓
transform(Int → String)
        ↓
String
```

You don't need to fully understand the advanced generic mechanics here yet.

I want you to notice that generic types can represent relationships between multiple types.

---

# This is one reason `map` is so powerful

You've already used:

```kotlin
numbers.map { it * 2 }
```

Conceptually, `map` is generic.

It doesn't say:

```text
List<Int> → List<Int>
```

It says something much more general:

```text
List<T>
   ↓
function T → R
   ↓
List<R>
```

So:

```kotlin
listOf(1, 2, 3)
    .map { it.toString() }
```

takes:

```text
List<Int>
```

and produces:

```text
List<String>
```

The transformation changes the element type.

This is a fantastic real-world example of generics expressing a relationship:

```text
input element type
        ↓
transformation
        ↓
output element type
```

---

# `filter` is different

Now compare:

```kotlin
numbers.filter { it > 10 }
```

It doesn't change the element type.

It takes:

```text
List<T>
```

and returns another:

```text
List<T>
```

So conceptually:

```text
filter
List<T> → List<T>

map
List<T> → List<R>
```

This is a very useful way to start understanding generic APIs.

Instead of memorizing every function independently, ask:

> **What type relationship does this operation preserve or change?**

---

# Generics and type safety

Let's see why this matters.

Suppose:

```kotlin
val names: MutableList<String> = mutableListOf()

names.add("Alex")
```

Fine.

But:

```kotlin
names.add(42)
```

doesn't compile.

Why?

Because the generic type says:

```text
MutableList<String>
```

not:

```text
MutableList<Anything>
```

The generic parameter is helping the compiler enforce what belongs in the collection.

That is one of the simplest and most important examples of generics providing safety.

---

# What happens if we used `MutableList<Any>`?

Then:

```kotlin
val list: MutableList<Any> = mutableListOf()

list.add("Alex")
list.add(42)
list.add(true)
```

All valid.

But perhaps that's not what we wanted.

The type system is allowing a mixed bag because we explicitly told it:

```text
Any
```

Generics let us be precise:

```text
MutableList<String>
MutableList<Int>
MutableList<User>
```

The more specific type communicates and enforces more.

---

# A generic repository example

Let's make something slightly more realistic.

Imagine:

```kotlin
data class User(
    val id: Int,
    val name: String
)
```

and:

```kotlin
data class Product(
    val id: Int,
    val name: String
)
```

Suppose we want something that stores and retrieves objects.

We could write:

```kotlin
class UserRepository {
    private val users = mutableListOf<User>()

    fun add(user: User) {
        users.add(user)
    }

    fun getAll(): List<User> {
        return users
    }
}
```

Then we'd need:

```kotlin
class ProductRepository
```

and another:

```kotlin
class OrderRepository
```

and so on.

Maybe there's a common abstraction:

```kotlin
class Repository<T> {

    private val items = mutableListOf<T>()

    fun add(item: T) {
        items.add(item)
    }

    fun getAll(): List<T> {
        return items
    }
}
```

Now:

```kotlin
val users = Repository<User>()
val products = Repository<Product>()
```

Same implementation.

Different type.

This is exactly where generic abstractions become useful.

---

# But don't over-generalize

This is important.

You could create:

```kotlin
Repository<T>
```

and then start trying to make every repository in your application fit one giant abstraction.

That may be a bad design.

Generics give you the _ability_ to generalize.

They don't tell you whether generalization is actually a good idea.

This is becoming a recurring lesson in this course:

> **A powerful abstraction is not automatically a good abstraction.**

Use a generic when the commonality is real.

---

# Here's something subtle: generic type parameters aren't values

In:

```kotlin
class Box<T>
```

`T` isn't a variable holding a value.

It's a placeholder at the **type level**.

Compare:

```kotlin
val x = 10
```

Here `x` is a value-level variable.

But:

```kotlin
class Box<T>
```

has:

```text
T
```

as a type-level parameter.

So:

```text
x → value

T → type
```

This distinction becomes increasingly important as we go deeper.

---

# You can constrain a generic type

Now imagine we have:

```kotlin
fun <T> something(value: T)
```

We don't know anything about `T`.

It could be:

```text
String
Int
User
Banana
```

So we can't assume it has particular properties or methods.

For example, this isn't valid conceptually:

```kotlin
fun <T> printLength(value: T) {
    println(value.length)
}
```

Why?

Because `T` could be `Int`.

Kotlin doesn't know that `T` has a `length`.

So perhaps we want:

> "T must be a String or something compatible with a particular constraint."

We can express constraints.

For example:

```kotlin
fun <T : Number> double(value: T) {
    // ...
}
```

Now we are saying:

```text
T
must be a subtype of Number
```

This introduces a huge topic: **generic constraints**.

We're not going deep into it today because I want you to first understand unconstrained generics.

But notice the motivation:

```text
generic
→ accepts many types

constraint
→ accepts many types, but only from a meaningful family
```

That is going to be our next step.

---

# One more thing: type inference

You've already seen:

```kotlin
val box = Box(42)
```

without writing:

```kotlin
val box: Box<Int>
```

Why?

Kotlin can infer the type argument.

It sees:

```text
42 : Int
```

and therefore infers:

```text
T = Int
```

So:

```text
Box(42)
```

becomes:

```text
Box<Int>
```

This is called **type inference**.

You can always be explicit when you want:

```kotlin
val box: Box<Int> = Box(42)
```

but usually you don't need to be.

---

# Sometimes you need to specify the type argument

Consider a function where Kotlin can't infer the type from the arguments.

You can provide type arguments explicitly in the call:

```kotlin
val box = emptyBox<String>()
```

for a suitably defined generic function.

For example:

```kotlin
fun <T> emptyBox(): Box<T>? {
    return null
}
```

Then:

```kotlin
val box = emptyBox<String>()
```

tells Kotlin:

```text
T = String
```

This is something you'll see increasingly as generic code becomes more sophisticated.

---

# Generic syntax is going to get more complicated

Eventually you'll encounter things like:

```kotlin
<T : Number>
```

then:

```kotlin
<T : Comparable<T>>
```

then:

```kotlin
<out T>
```

and:

```kotlin
<in T>
```

and:

```kotlin
where T : ...
```

Those can look intimidating.

Don't worry.

We're not going to throw all of that at you now.

First I want this idea to become natural:

> **A generic parameter is a placeholder for a type, allowing us to write reusable code while preserving type information.**

Everything else builds on that.

---

# One more connection to our earlier lessons

Remember functions as values?

You saw:

```kotlin
map { ... }
```

and:

```kotlin
filter { ... }
```

Remember data classes?

You used:

```kotlin
List<User>
```

Remember sealed types?

You modeled:

```kotlin
Result
```

Generics are the mechanism that lets these ideas scale.

Imagine:

```kotlin
sealed interface Result<out T>
```

Eventually you could have:

```text
Result<User>
Result<Post>
Result<String>
```

Now you have:

```text
one conceptual result model
+
different result data types
```

We're not going into `out` yet—that is a later lesson—but I want you to see where this road leads.

---

# The mental model

Don't think:

> "Generics are angle brackets."

Think:

```text
I have an idea that doesn't depend on the exact type.
            ↓
I don't want to duplicate the code.
            ↓
But I still want the compiler to know the exact type.
            ↓
Use a type parameter.
```

That's the reason.

---

# Your practice

I don't want you to memorize generic syntax today.

I want you to test whether you've understood what the type parameter is doing.

### 1

What do these mean?

```kotlin
Box<Int>
Box<String>
Box<User>
```

Explain them in plain language.

---

### 2

Why is this generally safer:

```kotlin
List<String>
```

than:

```kotlin
List<Any>
```

when you only want strings?

---

### 3

What's the real difference between:

```kotlin
class Box<T>
```

and:

```kotlin
class Box(
    val value: Any
)
```

?

Don't answer "one uses generics." Explain what information is preserved.

---

# Coding exercise

Write:

```kotlin
class Box<T>(
    val value: T
)
```

Then create boxes containing:

```text
an Int
a String
a Boolean
a data class of your own
```

For each one, try to access a property/method that belongs specifically to its actual type.

For example, with a `String` box:

```kotlin
box.value.length
```

Notice that Kotlin knows the specific type.

---

# Then build this

```kotlin
fun <T> first(list: List<T>): T
```

It should return the first element.

Test it with:

```kotlin
listOf(10, 20, 30)
```

and:

```kotlin
listOf("A", "B", "C")
```

The interesting thing isn't the implementation.

It's observing that **one function works with both lists while still returning `Int` in one case and `String` in the other.**

---

# One final thought

You've just crossed an important conceptual boundary.

Before generics, you mostly thought about:

```text
What is the type of this value?
```

Now we're beginning to think about:

```text
How can I write one piece of code that works with many types
without throwing away the information about which type I'm actually using?
```

That's the question generics answer.

Next we'll dig into **generic constraints and type bounds**. That's where generics stop meaning simply "any type" and start becoming:

> "Any type, provided it satisfies these rules."

And that's when generic programming starts becoming genuinely powerful.