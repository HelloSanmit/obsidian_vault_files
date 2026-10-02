 
# Lesson 10 — Functions as Values and Lambdas

This is one of the biggest conceptual jumps in the course.

Until now, we've treated a function as something that you **declare** and then **call**:

```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}

val result = add(10, 20)
```

The mental model was:

```text
function exists
     ↓
call function
     ↓
get result
```

Kotlin lets us go one step further.

A function can itself be treated as a **value**.

That means we can:

- store a function in a variable,
    
- pass a function to another function,
    
- return a function from a function,
    
- create functions without giving them a normal name.
    

This is the foundation of **lambdas**, **higher-order functions**, and a huge amount of modern Kotlin code.

---

# 1. Start with something familiar

We know this:

```kotlin
fun double(number: Int): Int {
    return number * 2
}
```

The name:

```text
double
```

identifies a function.

Normally we call it:

```kotlin
double(5)
```

and get:

```text
10
```

But imagine we want to **store that behavior** somewhere.

Conceptually:

```text
"double this number"
       ↓
store that behavior
       ↓
give it a name
```

Kotlin allows this.

---

# 2. Function types

Before we can store functions, we need a way to describe their type.

We've seen types such as:

```kotlin
Int
String
Boolean
```

Now imagine a function that:

```text
takes an Int
and returns an Int
```

Kotlin represents that function type as:

```kotlin
(Int) -> Int
```

Read that as:

> A function that takes an `Int` and returns an `Int`.

So:

```text
(Int) -> Int
```

is a **function type**.

Some examples:

```text
() -> Unit
```

means:

> A function taking nothing and returning `Unit`.

```text
(Int) -> String
```

means:

> A function taking an `Int` and returning a `String`.

```text
(String, Int) -> Boolean
```

means:

> A function taking a `String` and an `Int`, returning a `Boolean`.

This is exactly analogous to saying:

```text
age : Int
```

except now we're describing the shape of a function.

---

# 3. Storing a function in a variable

Let's write:

```kotlin
fun double(number: Int): Int {
    return number * 2
}

fun main() {
    val operation: (Int) -> Int = ::double

    println(operation(5))
}
```

Output:

```text
10
```

There is a new thing here:

```kotlin
::double
```

This means:

> Give me a reference to the function `double`.

It does **not** mean:

> Call `double`.

That's an important distinction.

Compare:

```kotlin
double(5)
```

with:

```kotlin
::double
```

The first:

```text
calls the function
```

The second:

```text
refers to the function itself
```

So conceptually:

```text
::double
    ↓
function
    ↓
store in operation
```

Then:

```kotlin
operation(5)
```

calls that stored function.

---

# 4. Why is this useful?

At first you might think:

> Why would I bother doing this? I could just call `double(5)`.

Exactly.

The power appears when we **pass the function around**.

Suppose we create:

```kotlin
fun applyOperation(
    number: Int,
    operation: (Int) -> Int
): Int {
    return operation(number)
}
```

Now we can do:

```kotlin
val result = applyOperation(5, ::double)
```

Conceptually:

```text
5
+
"here is the operation to perform"
        ↓
applyOperation
        ↓
operation(5)
        ↓
double(5)
        ↓
10
```

We've separated:

> **the data**

from:

> **what should be done with the data**

That's a tremendously powerful idea.

---

# 5. Another operation

Suppose:

```kotlin
fun triple(number: Int): Int {
    return number * 3
}
```

Now:

```kotlin
fun main() {
    println(applyOperation(5, ::double))
    println(applyOperation(5, ::triple))
}
```

Output:

```text
10
15
```

The `applyOperation` function itself doesn't know whether it's doubling or tripling.

It simply says:

> Give me a number and a function that knows what to do with that number.

This is our first genuine example of a **higher-order function**.

---

# 6. What is a higher-order function?

A function is called a **higher-order function** if it does at least one of these things:

- takes a function as a parameter,
    
- returns a function.
    

For example:

```kotlin
fun applyOperation(
    number: Int,
    operation: (Int) -> Int
): Int {
    return operation(number)
}
```

takes a function:

```kotlin
operation: (Int) -> Int
```

so it's a higher-order function.

This might sound advanced, but you're going to use this idea constantly in Kotlin.

---

# 7. Now we meet lambdas

So far, we created named functions:

```kotlin
fun double(number: Int): Int {
    return number * 2
}
```

But what if we don't actually need to give the function a name?

We could write a **lambda**:

```kotlin
{ number: Int -> number * 2 }
```

This is a function value without a normal function declaration.

Read it as:

> Take an `Int` called `number`, and produce `number * 2`.

So:

```kotlin
{ number: Int -> number * 2 }
```

is a lambda expression.

---

# 8. Store the lambda

We can do:

```kotlin
val double = { number: Int -> number * 2 }
```

Now:

```kotlin
println(double(5))
```

prints:

```text
10
```

And:

```text
double
```

is now a variable referring to a function.

This may look surprisingly similar to:

```kotlin
val x = 10
```

Compare:

```kotlin
val x = 10
```

with:

```kotlin
val double = { number: Int -> number * 2 }
```

In the first case:

```text
x → value 10
```

In the second:

```text
double → function
```

That's the conceptual shift.

---

# 9. The syntax of a lambda

The basic shape is:

```kotlin
{ parameter -> expression }
```

For example:

```kotlin
{ x: Int -> x * 2 }
```

means:

```text
parameter:
x : Int

result:
x * 2
```

Another:

```kotlin
{ name: String -> "Hello, $name" }
```

means:

```text
String → String
```

Another:

```kotlin
{ x: Int -> x > 10 }
```

means:

```text
Int → Boolean
```

So lambdas have types just like ordinary functions.

---

# 10. You can explicitly state the function type

For example:

```kotlin
val double: (Int) -> Int = { number ->
    number * 2
}
```

Now the function type is explicitly written:

```text
(Int) -> Int
```

Because Kotlin already knows `number` must be an `Int`, we don't need to write `: Int` on the lambda parameter:

```kotlin
val double: (Int) -> Int = { number ->
    number * 2
}
```

This is one of the places where Kotlin's type inference makes lambdas cleaner.

---

# 11. A lambda can have multiple parameters

For example:

```kotlin
val add: (Int, Int) -> Int = { a, b ->
    a + b
}
```

Now:

```kotlin
println(add(10, 20))
```

produces:

```text
30
```

The function type is:

```text
(Int, Int) -> Int
```

Meaning:

```text
two Int inputs
       ↓
one Int output
```

---

# 12. Lambdas can return different types

For example:

```kotlin
val describe: (Int) -> String = { age ->
    "Age: $age"
}
```

Or:

```kotlin
val isEven: (Int) -> Boolean = { number ->
    number % 2 == 0
}
```

So function types are really describing the **shape of computation**:

```text
(Int) -> String
(Int) -> Boolean
(Int, Int) -> Int
```

This is a very useful way to think about them.

---

# 13. Lambdas don't need to be stored in variables

Remember:

```kotlin
fun applyOperation(
    number: Int,
    operation: (Int) -> Int
): Int {
    return operation(number)
}
```

We can directly pass a lambda:

```kotlin
val result = applyOperation(
    5,
    { number -> number * 2 }
)
```

So instead of:

```text
create named function
       ↓
take reference to function
       ↓
pass reference
```

we can simply say:

```text
"here is the behavior you should use"
```

right at the point where we need it.

That's one of the reasons lambdas are so useful.

---

# 14. Kotlin has a special syntax for this: trailing lambdas

Because passing lambdas is so common, Kotlin gives them special syntax.

Instead of:

```kotlin
applyOperation(
    5,
    { number -> number * 2 }
)
```

we can write:

```kotlin
applyOperation(5) { number ->
    number * 2
}
```

The lambda is moved outside the parentheses.

This is called **trailing lambda syntax**.

You'll see this everywhere in Kotlin.

And this:

```kotlin
applyOperation(5) { number ->
    number * 2
}
```

should mentally mean:

> Call `applyOperation` with `5` and give it this behavior.

---

# 15. Why trailing lambdas exist

Because this pattern:

```kotlin
someFunction(data) {
    // behavior
}
```

reads naturally as:

> Give this function some data, then tell it what to do.

That style is used heavily throughout Kotlin libraries.

And it leads to code like:

```kotlin
names.filter { name -> name.length > 5 }
```

which we'll soon be able to fully understand.

---

# 16. The mysterious `it`

You may have seen:

```kotlin
numbers.filter { it > 10 }
```

and wondered:

> What is `it`?!

Now we can explain it.

Suppose a function expects:

```text
(Int) -> Boolean
```

and you write:

```kotlin
{ number -> number > 10 }
```

When there is **one parameter**, Kotlin often allows you to omit its name and use the implicit name:

```text
it
```

So:

```kotlin
{ number -> number > 10 }
```

can become:

```kotlin
{ it > 10 }
```

Same basic idea.

The `it` is the lambda's single parameter.

---

# 17. Don't overuse `it`

This is one of those places where concise Kotlin can become unclear.

Compare:

```kotlin
users.filter { it.age >= 18 }
```

with:

```kotlin
users.filter { user -> user.age >= 18 }
```

The first is wonderfully concise when the meaning is obvious.

The second can sometimes be easier to understand, especially with nested lambdas:

```kotlin
users.filter { user ->
    user.orders.any { order ->
        order.total > 1000
    }
}
```

Now there are two `it`s if written implicitly, and that can become confusing.

So:

> `it` is convenient, not mandatory.

---

# 18. This brings us to collection operations

You've probably seen Kotlin code like:

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

val doubled = numbers.map { it * 2 }
```

Now we can finally understand what is happening.

`map` is a function that:

> Takes a collection and a function describing how to transform each element.

The lambda:

```kotlin
{ it * 2 }
```

describes the transformation.

So:

```text
1 → 2
2 → 4
3 → 6
4 → 8
5 → 10
```

resulting in:

```text
[2, 4, 6, 8, 10]
```

`map` is therefore a higher-order function.

It receives another function.

This is one of the main ways functional programming ideas appear in everyday Kotlin.

---

# 19. Let's understand `filter`

Consider:

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6)

val evenNumbers = numbers.filter { it % 2 == 0 }
```

The lambda:

```kotlin
{ it % 2 == 0 }
```

has the type:

```text
(Int) -> Boolean
```

For each number, it asks:

```text
"Should this element be kept?"
```

For example:

```text
1 → false
2 → true
3 → false
4 → true
5 → false
6 → true
```

Result:

```text
[2, 4, 6]
```

So now you've seen two different kinds of functions being passed around:

```text
map
→ describes a transformation

filter
→ describes a condition
```

That is the power of higher-order functions.

---

# 20. The deeper pattern

Compare these:

```kotlin
numbers.map { it * 2 }
```

and:

```kotlin
numbers.filter { it > 10 }
```

They both involve:

```text
collection
   +
behavior
```

But the behavior has a different meaning.

For `map`:

```text
element → transformed element
```

For `filter`:

```text
element → true/false
```

So their function types are conceptually:

```text
map:
(T) -> R

filter:
(T) -> Boolean
```

This is where generics will eventually enter the picture.

You don't need to understand `T` and `R` yet.

Just notice that **functions themselves have structure**.

---

# 21. Functions can return functions

So far we've passed functions into other functions.

The reverse is also possible.

For example:

```kotlin
fun createMultiplier(factor: Int): (Int) -> Int {
    return { number ->
        number * factor
    }
}
```

Now:

```kotlin
val double = createMultiplier(2)
val triple = createMultiplier(3)
```

Then:

```kotlin
println(double(5))
println(triple(5))
```

Output:

```text
10
15
```

Think about what's happening.

We call:

```text
createMultiplier(2)
```

and it returns a function:

```text
number → number × 2
```

Then:

```text
createMultiplier(3)
```

returns another function:

```text
number → number × 3
```

This is a huge conceptual step.

A function can **manufacture another function**.

---

# 22. But where did `factor` go?

This example is especially interesting:

```kotlin
fun createMultiplier(factor: Int): (Int) -> Int {
    return { number ->
        number * factor
    }
}
```

When we do:

```kotlin
val double = createMultiplier(2)
```

the returned lambda still knows about:

```text
factor = 2
```

Later, even though `createMultiplier` has finished, calling:

```kotlin
double(5)
```

still uses `2`.

This is called a **closure**.

A closure is essentially a function together with the surrounding values it has captured.

That's an advanced concept, but it's important enough to see now.

We'll return to closures later and look at exactly how they work.

---

# 23. Why are lambdas so powerful?

Because they allow you to separate:

```text
what to do
```

from:

```text
when / where to do it
```

Suppose we have:

```kotlin
fun repeatThreeTimes(action: () -> Unit) {
    action()
    action()
    action()
}
```

Then:

```kotlin
repeatThreeTimes {
    println("Hello")
}
```

produces:

```text
Hello
Hello
Hello
```

The `repeatThreeTimes` function doesn't care what the action is.

It just knows:

> I have something I can call.

This pattern is the basis for a tremendous number of APIs.

---

# 24. A function type is like a contract

Suppose we write:

```kotlin
fun execute(action: () -> Unit) {
    action()
}
```

The type:

```text
() -> Unit
```

is a contract.

It says:

> Give me a function that takes no arguments and produces no meaningful return value.

Therefore this is valid:

```kotlin
execute {
    println("Hello")
}
```

And this:

```kotlin
execute {
    println("Doing something")
}
```

also works.

But a function requiring an argument wouldn't match that type:

```text
(Int) -> Unit
```

is a different shape from:

```text
() -> Unit
```

This is similar to how:

```text
Int
```

and:

```text
String
```

are different types.

Function types are types too.

---

# 25. Functions don't have to be named to be useful

Consider:

```kotlin
numbers.map { it * 2 }
```

There is no function declaration like:

```kotlin
fun double(...)
```

The behavior is defined exactly where it is needed.

That can make code incredibly expressive.

Compare:

```kotlin
fun double(number: Int): Int {
    return number * 2
}

numbers.map(::double)
```

with:

```kotlin
numbers.map { it * 2 }
```

Both can be appropriate.

The second is especially convenient when the transformation is tiny and used only there.

The first makes more sense if the operation has a meaningful name or needs to be reused.

Again, we're looking for **clarity**, not "the shortest possible Kotlin."

---

# 26. Lambda vs named function

A useful rule of thumb:

### Named function

Prefer when:

- the operation has an important concept/name,
    
- it is reused,
    
- it is complex,
    
- it deserves independent testing,
    
- the implementation is easier to understand separately.
    

### Lambda

Prefer when:

- the behavior is small,
    
- it is used once,
    
- the meaning is obvious from context,
    
- you're configuring another function.
    

For example:

```kotlin
numbers.map { it * 2 }
```

is natural.

Whereas a complicated twenty-line lambda nested inside another function can become unpleasant.

---

# 27. One more piece of syntax: function references

Suppose:

```kotlin
fun isEven(number: Int): Boolean {
    return number % 2 == 0
}
```

We can pass the function itself using:

```kotlin
::isEven
```

For example:

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6)

val evens = numbers.filter(::isEven)
```

That's conceptually similar to:

```kotlin
numbers.filter { it % 2 == 0 }
```

The difference is that the first uses an existing named function, while the second creates a lambda inline.

Both express:

```text
Int → Boolean
```

---

# 28. A subtle distinction: calling vs referencing

Please pay attention to this:

```kotlin
isEven(4)
```

means:

> Execute `isEven` now with `4`.

Whereas:

```kotlin
::isEven
```

means:

> Give me the function itself.

This distinction is fundamental.

Just like:

```text
recipe
```

is different from:

```text
cook(recipe)
```

The first represents the thing.

The second performs something using it.

---

# 29. Let's build a tiny "calculator engine"

Now we can use functions as data.

```kotlin
fun calculate(
    a: Double,
    b: Double,
    operation: (Double, Double) -> Double
): Double {
    return operation(a, b)
}
```

We can pass different operations:

```kotlin
val add = { a: Double, b: Double -> a + b }
val multiply = { a: Double, b: Double -> a * b }
val subtract = { a: Double, b: Double -> a - b }
```

Then:

```kotlin
println(calculate(10.0, 5.0, add))
println(calculate(10.0, 5.0, multiply))
println(calculate(10.0, 5.0, subtract))
```

Results:

```text
15.0
50.0
5.0
```

The `calculate` function doesn't know the specific operation.

It receives behavior from the caller.

This is **dependency through behavior**, and it becomes extremely useful in software design.

---

# 30. This is the idea behind callbacks

A callback is essentially:

> A function you give to another piece of code so that it can call you back later or at an appropriate point.

For example:

```kotlin
fun doSomething(onComplete: () -> Unit) {
    println("Doing work...")
    onComplete()
}
```

Then:

```kotlin
doSomething {
    println("Finished!")
}
```

Output:

```text
Doing work...
Finished!
```

This idea appears everywhere:

- UI programming
    
- event handling
    
- asynchronous programming
    
- networking
    
- coroutines
    
- libraries
    

Once we reach Android and coroutines, you'll see this concept repeatedly.

---

# 31. Don't worry about memorizing every lambda syntax variation

At this stage, I'd rather you understand:

```text
function
   ↓
can be represented as a value
   ↓
has a function type
   ↓
can be passed around
   ↓
can be called
```

and:

```text
lambda
   ↓
unnamed function value
```

The syntax will become automatic through practice.

---

# 32. Let's connect this to everything we've already learned

Remember our basic model:

```text
value
 ↓
type
```

Now functions fit into the same model:

```text
function value
      ↓
function type
```

For example:

```text
lambda:
{ x -> x * 2 }

type:
(Int) -> Int
```

Then:

```text
function value
      ↓
passed to another function
      ↓
called
      ↓
produces value
```

So functions have become part of the same world as ordinary data.

That's the conceptual leap.

---

# 33. Your exercises

### Exercise 1

Create a lambda that doubles an integer:

```text
(Int) -> Int
```

Store it in a variable and call it with `10`.

---

### Exercise 2

Create a lambda:

```text
(Int) -> Boolean
```

that returns whether a number is positive.

---

### Exercise 3

Write:

```text
apply
```

that takes:

```text
an Int
+
a function (Int) -> Int
```

and returns the result.

Then use it with:

- double
    
- triple
    
- square
    

---

### Exercise 4

Create:

```text
createAdder(amount)
```

which returns a function that adds `amount` to its input.

For example, conceptually:

```text
createAdder(10)
      ↓
function that adds 10

thatFunction(5)
      ↓
15
```

This one introduces closures, so take your time with it.

---

### Exercise 5

Given:

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
```

use `filter` with a lambda to produce:

```text
[2, 4, 6, 8, 10]
```

Then use `map` to turn those into:

```text
[4, 8, 12, 16, 20]
```

Try to understand what function you're giving to `filter` and what function you're giving to `map`.

---

# Challenge

Write:

```kotlin
fun transform(
    numbers: List<Int>,
    operation: (Int) -> Int
): List<Int>
```

The function should apply `operation` to every number and return the transformed list.

Then make these work:

```text
transform(numbers, double)
transform(numbers, square)
transform(numbers, triple)
```

Don't worry yet about implementing this using `map`. Try to think about what `transform` itself has to do.

That exercise will reinforce the central idea of this lesson:

> **Behavior can be passed around just like data.**

---

# The mental model I want you to leave with

Before this lesson, you probably thought of functions primarily as:

```text
define function
      ↓
call function
      ↓
get result
```

Now expand it:

```text
                         Function
                            │
              ┌─────────────┼─────────────┐
              ↓             ↓             ↓
           define        store         pass
                                          │
                                          ↓
                                      another
                                      function
                                          │
                                          ↓
                                        call
                                          │
                                          ↓
                                       result
```

And lambdas give us a compact way to create function values:

```kotlin
{ x -> x * 2 }
```

This is going to unlock a lot of Kotlin syntax that initially looks strange.

For example:

```kotlin
numbers
    .filter { it > 10 }
    .map { it * 2 }
```

is no longer mysterious.

It's essentially:

```text
numbers
  ↓
give filter a function
  ↓
keep elements satisfying that function
  ↓
give map another function
  ↓
transform each remaining element
```

And that's exactly where we're going next.

# Next: Collections — Lists, Sets, Maps, Mutability, and `map` / `filter` / `fold`

We've already touched collections and higher-order functions separately. Next we'll bring them together properly and learn **how Kotlin represents groups of data and how to process them cleanly**.