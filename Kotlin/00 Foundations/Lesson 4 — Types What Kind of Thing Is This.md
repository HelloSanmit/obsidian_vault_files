
# Lesson 4 — Types: What Kind of Thing Is This?

Last lesson we wrote:

```kotlin
val age = 25
```

and said that Kotlin can infer the type of `age`.

Today I want to slow down and understand **what a type actually is**, because types are one of the foundations of programming.

You'll see types everywhere in Kotlin:

```kotlin
Int
Double
String
Boolean
List<String>
User
Result<T>
```

At first they may look like labels attached to variables.

They're much more important than that.

---

## 1. What is a type?

Imagine I tell you:

> Here is the value `25`.

What can you do with it?

You might add something:

```text
25 + 10
```

multiply it:

```text
25 × 3
```

compare it:

```text
25 > 18
```

But now imagine I tell you:

> Here is the value `"Alice"`.

You can't meaningfully do:

```text
"Alice" × 3
```

At least not in the ordinary mathematical sense.

The **type** of a value tells the language what kind of thing that value is and, consequently, what operations are meaningful on it.

For example:

```text
25
↓
Int
```

and:

```text
"Alice"
↓
String
```

and:

```text
true
↓
Boolean
```

So a useful first mental model is:

> **A type describes the kind of value you're dealing with and the operations that make sense for it.**

---

# 2. Kotlin has different basic types

Some of the first ones you'll encounter are:

```text
Int       whole numbers
Long      larger whole numbers
Double    decimal numbers
Float     decimal numbers
Boolean   true / false
Char      a single character
String    text
```

Let's look at them one at a time.

---

# 3. `Int`

An `Int` represents an integer value.

Examples:

```kotlin
val age: Int = 25
val score: Int = 100
val temperature: Int = -5
```

These are whole numbers.

No decimal portion:

```text
-10
0
1
25
100
```

When you write:

```kotlin
val age = 25
```

Kotlin will normally infer:

```kotlin
Int
```

So:

```kotlin
val age = 25
```

is essentially an `Int` variable.

---

# 4. `Double`

A `Double` represents a floating-point decimal number.

For example:

```kotlin
val price: Double = 99.99
val pi: Double = 3.14159
```

And Kotlin can infer it:

```kotlin
val price = 99.99
```

So:

```text
25
```

and:

```text
25.0
```

are not the same type.

Conceptually:

```text
25     → Int
25.0   → Double
```

That distinction matters.

---

# 5. This is where programming starts feeling different from normal mathematics

In mathematics, you might comfortably think:

```text
25 = 25.0
```

They're numerically equal.

But in a programming language, there are different **types** involved.

So:

```kotlin
val a = 25
val b = 25.0
```

means:

```text
a → Int
b → Double
```

The language doesn't just think:

> "They're both numbers, so everything should work automatically."

Kotlin is stricter than that.

And this is intentional.

---

# 6. Why does Kotlin care so much about types?

Because without types, programs become ambiguous and error-prone.

Imagine:

```kotlin
val a = 10
val b = 2
```

Kotlin knows these are integers.

So:

```kotlin
a / b
```

has integer division semantics.

But if:

```kotlin
val a = 10.0
val b = 2.0
```

then:

```kotlin
a / b
```

is floating-point division.

The types help determine what the operation means.

This is one of the central ideas of statically typed programming:

> **The compiler knows a lot about the kinds of values your program is working with before the program runs.**

We'll eventually go much deeper into why this is useful.

---

# 7. Let's investigate the famous `10 / 3`

Remember the question from Lesson 2.

What do you think this prints?

```kotlin
fun main() {
    println(10 / 3)
}
```

If you're thinking like a mathematician:

```text
3.333333...
```

But in Kotlin, `10` and `3` are `Int`s.

Therefore this is integer division.

The result is:

```text
3
```

Not:

```text
3.333333...
```

The fractional part is discarded.

So:

```kotlin
println(10 / 3)
```

prints:

```text
3
```

Whereas:

```kotlin
println(10.0 / 3.0)
```

produces a decimal result.

This is a very important lesson:

> **The operation is influenced by the types of the values involved.**

---

# 8. Try these mentally

What does each produce?

```kotlin
println(10 / 3)
println(10.0 / 3.0)
println(10 / 3.0)
```

The first is integer division.

The second is floating-point division.

The third involves an `Int` and a `Double`, so Kotlin has to determine a compatible operation.

Unlike some languages, Kotlin does not generally perform arbitrary implicit numeric widening for you.

For example, you shouldn't think:

> "Kotlin will just convert the `Int` to `Double` automatically everywhere."

It is deliberately stricter.

This becomes particularly important when assigning values to explicitly typed variables.

---

# 9. Kotlin does not casually convert numeric types

Suppose:

```kotlin
val x: Int = 10
```

and:

```kotlin
val y: Double = x
```

This won't work simply because mathematically `10` can be represented as `10.0`.

You need an explicit conversion:

```kotlin
val y: Double = x.toDouble()
```

Similarly:

```kotlin
val number = 10

val decimal = number.toDouble()
```

Now the conversion is explicit.

Kotlin provides functions such as:

```kotlin
toInt()
toLong()
toDouble()
toFloat()
toShort()
toByte()
```

So:

```kotlin
val x = 10
val y = x.toDouble()
```

means:

```text
x
↓
Int
↓
explicit conversion
↓
Double
↓
y
```

I want you to notice something philosophical here.

Kotlin is essentially saying:

> "If you're asking me to change the representation/type of this value, be explicit about it."

That can prevent accidental conversions.

---

# 10. `Long`

Suppose you need a whole number larger than the range of a normal `Int`.

Kotlin provides `Long`.

You can write:

```kotlin
val population: Long = 8_000_000_000
```

Notice the underscores.

Kotlin allows numeric separators to make large numbers easier to read:

```kotlin
8_000_000_000
```

instead of:

```kotlin
8000000000
```

They don't change the numeric value.

Another way to create a `Long` literal is to use `L`:

```kotlin
val distance = 8_000_000_000L
```

The `L` tells Kotlin that this literal should be treated as a `Long`.

For ordinary everyday integers, you will most often use `Int`.

Don't start using `Long` everywhere just because it can hold larger values.

---

# 11. What about `Float`?

`Float` is also a floating-point type.

For example:

```kotlin
val temperature: Float = 36.5f
```

Notice the `f`.

Kotlin normally treats decimal literals as `Double`, so when you want a `Float` literal you write:

```kotlin
36.5f
```

rather than:

```kotlin
36.5
```

So:

```text
36.5
↓
Double
```

while:

```text
36.5f
↓
Float
```

For many general-purpose calculations, `Double` is the more common choice.

We'll eventually discuss why floating-point numbers have surprising precision behavior, because that's an important practical topic.

---

# 12. `Boolean`

A Boolean has exactly two logical values:

```kotlin
true
false
```

For example:

```kotlin
val isLoggedIn = true
val hasPermission = false
```

Its type is:

```text
Boolean
```

This becomes extremely important when we study conditions:

```kotlin
if (isLoggedIn) {
    println("Welcome")
}
```

We'll get to `if` shortly.

For now, understand:

```text
Boolean = true or false
```

---

# 13. `Char`

A `Char` represents a single character.

For example:

```kotlin
val grade: Char = 'A'
val initial: Char = 'J'
```

Notice the quotes:

```text
'A'
```

not:

```text
"A"
```

That's important.

Compare:

```kotlin
val letter = 'A'
val word = "A"
```

The first is a `Char`.

The second is a `String`.

So:

```text
'A'   → Char
"A"   → String
```

Even though they look extremely similar.

---

# 14. `String`

A `String` represents text.

For example:

```kotlin
val name = "Alice"
val city = "Pune"
val message = "Hello, Kotlin!"
```

Strings use double quotes:

```kotlin
"Hello"
```

while a `Char` uses single quotes:

```kotlin
'H'
```

This distinction will matter constantly when we work with text.

---

# 15. How does Kotlin know the type without us telling it?

This is **type inference**.

For example:

```kotlin
val age = 25
```

Kotlin sees the literal `25` and infers an appropriate type.

Similarly:

```kotlin
val price = 19.99
```

is inferred as a decimal numeric type.

And:

```kotlin
val name = "Alice"
```

is inferred as `String`.

This allows code like:

```kotlin
val name = "Alice"
val age = 25
val active = true
```

instead of:

```kotlin
val name: String = "Alice"
val age: Int = 25
val active: Boolean = true
```

The shorter version is often preferred when the type is obvious.

---

# 16. But the type still exists

This is important.

When you write:

```kotlin
val age = 25
```

Kotlin isn't saying:

> "This variable has no type."

Quite the opposite.

The variable has a type.

Kotlin simply **inferred it for you**.

Conceptually:

```text
val age = 25
       ↓
age : Int
```

So type inference is not the absence of types.

It's the compiler figuring out the type for you.

---

# 17. A good experiment

Write:

```kotlin
fun main() {
    val age = 25
    val price = 99.99
    val name = "Alice"
    val active = true
    val grade = 'A'
}
```

Then hover over each variable in IntelliJ.

The IDE will show you the inferred type.

That's a good way to start developing intuition for what Kotlin sees.

You can also explicitly annotate them:

```kotlin
val age: Int = 25
val price: Double = 99.99
val name: String = "Alice"
val active: Boolean = true
val grade: Char = 'A'
```

Both forms are valid.

---

# 18. Types affect which operations you can perform

This is perhaps the most important practical point.

Consider:

```kotlin
val age = 25
```

Since `age` is an `Int`, operations like these make sense:

```kotlin
age + 5
age - 2
age * 3
age / 2
```

But now:

```kotlin
val name = "Alice"
```

You wouldn't expect:

```kotlin
name * 5
```

to have the same meaning.

Instead, strings have operations appropriate to text.

For example:

```kotlin
println(name.length)
```

gives the length of the string.

So the type gives us a vocabulary of meaningful operations.

We'll explore this idea much more when we get into classes and objects.

---

# 19. An analogy

Think of types somewhat like categories of tools.

Suppose you have:

```text
Hammer
Screwdriver
Saw
```

You wouldn't say:

> "They're all tools, therefore every operation should work on every tool."

Each has capabilities appropriate to what it is.

Types play a similar conceptual role.

An `Int` has numeric behavior.

A `String` has text behavior.

A `Boolean` has logical behavior.

Later you'll create your own types:

```kotlin
class User
class BankAccount
class Product
```

and those types will have their own data and behavior.

So types aren't just "containers for values."

They become a way of **modeling the world inside a program**.

That's going to become a major theme later.

---

# 20. A subtle point about `Int`

An `Int` doesn't mean:

> "A number."

It means something more specific:

> **An integer value within the range representable by Kotlin's `Int` type.**

Likewise:

```text
Double ≠ "any possible decimal number"
```

It represents a floating-point representation with specific characteristics and limits.

We'll eventually discuss:

- range
    
- overflow
    
- precision
    
- floating-point representation
    

because these become relevant when writing robust software.

---

# 21. Integer overflow — a glimpse

Consider:

```kotlin
var x = Int.MAX_VALUE
println(x)
```

`Int.MAX_VALUE` is the largest value representable by an `Int`.

Now imagine adding one.

```kotlin
x += 1
```

The result doesn't become an infinitely larger `Int`.

The representation has limits.

We'll investigate exactly what happens and why when we study numeric behavior more deeply.

For now, the lesson is:

> **Types have constraints.**

A type isn't an unlimited mathematical abstraction.

---

# 22. Why explicit typing can be useful

Suppose you write:

```kotlin
val temperature = 25
```

It's obvious what the type should be.

But imagine:

```kotlin
val amount: Double = 25.0
```

Here the explicit type may communicate something important about the program's intent.

Especially in APIs, function parameters, return types and domain models, explicit types are often valuable.

So don't turn type inference into:

> "Never write types."

The actual principle is closer to:

> **Let Kotlin infer types when that improves readability, and write types explicitly when they communicate something useful.**

We'll develop your judgment about this over time.

---

# 23. Let's combine types with variables

Here's a tiny program:

```kotlin
fun main() {
    val name: String = "Alice"
    val age: Int = 25
    val height: Double = 1.68
    val isStudent: Boolean = true
    val grade: Char = 'A'

    println(name)
    println(age)
    println(height)
    println(isStudent)
    println(grade)
}
```

We've now stored several different kinds of information.

Conceptually:

```text
name       → String
age        → Int
height     → Double
isStudent  → Boolean
grade      → Char
```

This is the beginning of **modeling data**.

Soon we'll use these different types together to make decisions and build more interesting programs.

---

# 24. Something I want you to stop doing

Don't think of types as something you need to memorize for an exam.

Instead, when you see:

```kotlin
val age = 25
```

start developing the instinct:

> `age` is an `Int`.

When you see:

```kotlin
val price = 19.99
```

think:

> `price` is a `Double`.

And when you see:

```kotlin
val loggedIn = true
```

think:

> `loggedIn` is a `Boolean`.

Eventually this becomes automatic.

That's when the type system starts working for you instead of feeling like extra syntax.

---

# 25. The important difference between `Int` and `Double`

Let's make this concrete.

```kotlin
val a = 10
val b = 3

println(a / b)
```

→ integer division.

Now:

```kotlin
val a = 10.0
val b = 3.0

println(a / b)
```

→ floating-point division.

So the type isn't just metadata sitting in the background.

It can directly affect what your program **does**.

That's a major idea.

---

# 26. Your exercise

Write a program containing these variables:

```text
your name
your age
your height
whether you are a student
your first initial
```

Use the appropriate Kotlin types.

Then print them.

After that, try these experiments.

### Experiment A

Write:

```kotlin
val age = 25
val ageAsDouble = age.toDouble()
```

Then print both.

### Experiment B

Try to assign an `Int` directly to a `Double`:

```kotlin
val x: Int = 10
val y: Double = x
```

Read the compiler error carefully.

Then fix it explicitly.

### Experiment C

Predict the output:

```kotlin
fun main() {
    println(10 / 3)
    println(10.0 / 3.0)
    println(10 + 3.5)
}
```

Don't run it until you've made your predictions.

### Experiment D

Try:

```kotlin
val letter = 'A'
val text = "A"
```

Then inspect their inferred types in IntelliJ.

Ask yourself:

> Why did Kotlin choose different types even though both contain the letter A?

---

# One final mental model

We're now starting to build a hierarchy:

```text
Value
  ↓
has a type
  ↓
Type determines meaningful operations
  ↓
Operations produce new values
```

For example:

```text
10
 ↓
Int
 ↓
10 + 5
 ↓
15
 ↓
Int
```

Or:

```text
"Alice"
   ↓
String
   ↓
length
   ↓
5
   ↓
Int
```

That's a very powerful way to think about programming.

A value has a type.

That type determines what you can sensibly do with the value.

Those operations produce other values, which themselves have types.

And that chain of values, operations and decisions is essentially what our programs are made from.

Next, we'll take a major step forward and look at **operators and expressions properly**—including arithmetic, comparison, equality, ranges, Boolean logic, and the crucial difference between `=` and `==`.