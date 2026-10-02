
# Lesson 9 — Functions More Deeply: Scope, Parameters, Return Values, and Side Effects

We understand the basic idea of a function now:

```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}
```

It takes input, performs some work, and returns a result.

But there are several questions hiding underneath that basic definition:

- Where do `a` and `b` actually exist?
    
- Why can't another function access them?
    
- What happens to local variables after the function finishes?
    
- What's the difference between returning a value and changing something outside the function?
    
- When should a function take parameters rather than relying on variables outside itself?
    
- When should we use default or named arguments?
    

These questions are more important than simply knowing how to write `fun`.

---

## 1. Parameters belong to the function call

Take this:

```kotlin
fun greet(name: String) {
    println("Hello, $name")
}
```

and:

```kotlin
fun main() {
    greet("Alice")
}
```

When `greet("Alice")` executes, you can conceptually imagine:

```text
greet is called
      ↓
name gets "Alice"
      ↓
function body runs
      ↓
"Hello, Alice" is printed
      ↓
function finishes
```

Now consider:

```kotlin
fun greet(name: String) {
    println(name)
}

fun main() {
    greet("Alice")
    println(name)
}
```

The second `println(name)` is invalid.

Why?

Because `name` belongs to the scope of `greet`.

It isn't a globally available name.

This brings us to one of the most important concepts in programming:

# Scope

---

# 2. What is scope?

**Scope is the region of a program in which a particular name can be accessed.**

For example:

```kotlin
fun main() {
    val age = 25

    println(age)
}
```

`age` can be used inside `main`.

But:

```kotlin
fun main() {
    val age = 25
}

fun anotherFunction() {
    println(age)
}
```

doesn't work.

`age` was declared inside `main`, so its scope doesn't extend into `anotherFunction`.

Think of it like a boundary:

```text
main
┌────────────────────────────┐
│                            │
│    age                     │
│                            │
│    println(age)            │
│                            │
└────────────────────────────┘

anotherFunction
┌────────────────────────────┐
│                            │
│    age  ← doesn't exist    │
│                            │
└────────────────────────────┘
```

The variable exists conceptually for the execution of that scope, but the **name is accessible only where its scope allows**.

We'll later discover that scope and object lifetime are related but not identical concepts. For now, don't confuse:

> "I can't access this name here"

with:

> "The underlying memory definitely no longer exists."

Those are different questions.

---

# 3. Parameters have local scope

Look at:

```kotlin
fun add(a: Int, b: Int): Int {
    val result = a + b
    return result
}
```

There are three local names here:

```text
a
b
result
```

All belong to the function's local scope.

Outside:

```kotlin
fun main() {
    val answer = add(10, 20)

    println(answer)
}
```

we can't do:

```kotlin
println(a)
```

because `a` isn't part of `main`.

This separation is useful.

Imagine every function could freely access every variable in every other function. A large program would become extremely difficult to reason about.

Local scope gives functions a degree of **isolation**.

---

# 4. Scope can also exist inside blocks

You don't need a function to create a scope.

For example:

```kotlin
fun main() {
    if (true) {
        val message = "Hello"
        println(message)
    }

    println(message)
}
```

The second `println(message)` is invalid.

Why?

Because `message` was declared inside the `if` block.

Its scope is limited to that block.

So you can think of braces as creating nested regions:

```text
main scope
│
├── age
│
└── if scope
    │
    └── message
```

`message` exists within the nested scope but isn't available in the outer scope.

---

# 5. Inner scopes can usually access outer variables

Now reverse the situation:

```kotlin
fun main() {
    val age = 25

    if (age >= 18) {
        println(age)
    }
}
```

This works.

Why?

Because the `if` block is nested inside `main`, and the nested scope can access names from the surrounding scope.

Conceptually:

```text
main scope
│
│ age
│
└── if scope
      │
      └── can see age
```

So a useful rule is:

> An inner scope can generally see names from its enclosing scopes, but an outer scope cannot see names declared only inside an inner scope.

We'll encounter increasingly sophisticated versions of this idea with lambdas and closures.

---

# 6. Functions should usually receive what they need

Compare these two designs.

### Version A

```kotlin
var price = 100
var quantity = 3

fun calculateTotal(): Int {
    return price * quantity
}
```

The function secretly depends on variables outside itself.

### Version B

```kotlin
fun calculateTotal(price: Int, quantity: Int): Int {
    return price * quantity
}
```

Now the function explicitly declares its dependencies.

This is generally easier to understand.

When you see:

```kotlin
calculateTotal(price, quantity)
```

you know exactly what information the function needs.

Whereas with:

```kotlin
calculateTotal()
```

you'd have to inspect the function's implementation to discover which external variables it is using.

This leads to an important engineering principle:

> **Prefer making a function's important inputs explicit.**

There are exceptions, and we'll eventually discuss dependency injection, object state, closures, and other cases. But this is an excellent default.

---

# 7. Functions become easier to test when inputs are explicit

Suppose:

```kotlin
fun calculateTotal(price: Double, quantity: Int): Double {
    return price * quantity
}
```

We can test it directly:

```kotlin
calculateTotal(100.0, 3)
calculateTotal(50.0, 2)
calculateTotal(0.0, 10)
```

The function's result depends on the arguments.

That's useful because we don't need to construct an entire application just to test the calculation.

Compare that with a function that secretly reads global state, database state, system time, and environment variables.

Such a function becomes much harder to test.

We'll return to this idea when we reach testing and architecture.

---

# 8. Return value vs side effect

Now we get to an important distinction.

Consider:

```kotlin
fun square(x: Int): Int {
    return x * x
}
```

This function produces a value:

```text
square(5)
   ↓
25
```

It doesn't directly change anything outside itself.

Now compare:

```kotlin
var score = 0

fun increaseScore() {
    score += 10
}
```

This function doesn't return a meaningful value.

Instead, it **changes state outside itself**.

That change is called a **side effect**.

---

# 9. What counts as a side effect?

A function has a side effect when calling it does something observable beyond simply computing and returning its result.

Examples include:

```text
changing a variable outside the function
printing to the console
writing to a file
sending a network request
updating a database
logging
changing UI state
```

For example:

```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}
```

is primarily a computation.

Whereas:

```kotlin
fun printSum(a: Int, b: Int) {
    println(a + b)
}
```

has an observable effect: it writes to the console.

And:

```kotlin
fun increaseScore() {
    score += 10
}
```

changes external state.

This distinction will become very important once we get to functional programming, coroutines, testing, and architecture.

---

# 10. Why should we care about side effects?

Because pure calculations are usually easier to reason about.

Imagine:

```kotlin
fun multiply(a: Int, b: Int): Int {
    return a * b
}
```

Whenever you call:

```kotlin
multiply(5, 3)
```

you know what the answer is:

```text
15
```

It doesn't depend on:

- the current time
    
- a global variable
    
- what another function did
    
- a database
    
- a network connection
    

That predictability is valuable.

Now imagine:

```kotlin
fun calculateDiscount(price: Double): Double {
    return if (isPremiumUser) {
        price * 0.20
    } else {
        price * 0.05
    }
}
```

Where does `isPremiumUser` come from?

If it's global state, the function's result depends on something hidden.

Compare:

```kotlin
fun calculateDiscount(
    price: Double,
    isPremiumUser: Boolean
): Double {
    return if (isPremiumUser) {
        price * 0.20
    } else {
        price * 0.05
    }
}
```

Now the dependency is explicit.

Much easier to reason about.

---

# 11. This doesn't mean "side effects are bad"

They aren't.

A real application needs side effects.

Eventually you'll need to:

```text
read a database
send an HTTP request
save a file
display something
log an error
update UI
```

Those are inherently effectful operations.

The important engineering skill is learning to **control and isolate side effects**, rather than pretending they don't exist.

For example:

```text
Pure logic
   ↓
calculate result
   ↓
Effectful layer
   ↓
save/send/display result
```

We'll eventually build architectures around this idea.

---

# 12. Parameters can have default values

Now let's return to function syntax.

Suppose:

```kotlin
fun greet(name: String, greeting: String) {
    println("$greeting, $name!")
}
```

Every call needs both arguments:

```kotlin
greet("Alice", "Hello")
```

What if `"Hello"` is almost always what we want?

We can provide a default:

```kotlin
fun greet(name: String, greeting: String = "Hello") {
    println("$greeting, $name!")
}
```

Now:

```kotlin
greet("Alice")
```

works.

And:

```kotlin
greet("Alice", "Good morning")
```

also works.

So the second parameter has a default value.

Conceptually:

```text
greet("Alice")
        ↓
greeting = "Hello"

greet("Alice", "Hi")
        ↓
greeting = "Hi"
```

---

# 13. Default arguments reduce unnecessary overloads

This matters especially when designing APIs.

Imagine you wanted:

```kotlin
greet("Alice")
greet("Alice", "Hello")
```

In some languages, one common solution is to create multiple versions of the function.

Kotlin's default arguments often let you keep one function:

```kotlin
fun greet(
    name: String,
    greeting: String = "Hello"
) {
    println("$greeting, $name!")
}
```

This is a small feature, but you'll see it constantly in Kotlin code.

---

# 14. Named arguments

Suppose we have:

```kotlin
fun createUser(
    name: String,
    age: Int,
    active: Boolean
) {
    // ...
}
```

A positional call:

```kotlin
createUser("Alice", 25, true)
```

works.

But:

```kotlin
createUser(
    name = "Alice",
    age = 25,
    active = true
)
```

makes the meaning explicit.

Named arguments become particularly useful when:

- there are many arguments,
    
- several arguments have the same type,
    
- some arguments are optional,
    
- the call would otherwise be hard to understand.
    

For example:

```kotlin
drawRectangle(
    width = 200,
    height = 100,
    filled = true
)
```

is immediately readable.

---

# 15. You can combine named and default arguments

Suppose:

```kotlin
fun connect(
    host: String,
    port: Int = 80,
    secure: Boolean = false
) {
    // ...
}
```

You can write:

```kotlin
connect("example.com")
```

or:

```kotlin
connect(
    host = "example.com",
    secure = true
)
```

Here the omitted `port` uses its default.

This gives function callers flexibility without forcing them to provide irrelevant information.

---

# 16. Parameter order can influence API design

Consider:

```kotlin
fun createUser(
    name: String,
    age: Int,
    city: String = "Unknown",
    active: Boolean = true
)
```

A call can be:

```kotlin
createUser("Alice", 25)
```

or:

```kotlin
createUser(
    name = "Alice",
    age = 25,
    city = "Pune",
    active = false
)
```

You'll eventually learn conventions for designing good APIs, because function signatures become part of how other programmers interact with your code.

---

# 17. Functions can have no parameters

We've already seen:

```kotlin
fun sayHello() {
    println("Hello")
}
```

The parentheses are still important:

```kotlin
sayHello()
```

The empty `()` means:

> This function accepts no arguments.

Later you'll see functions with:

```text
()
(x)
(x, y)
(x, y, z)
```

and functions with default parameters, varargs, generic parameters, receiver parameters and much more.

But the basic idea remains:

```text
input → function → output
```

---

# 18. A function can return another function later

We're not studying this yet, but I want you to see where we're heading.

Kotlin treats functions as values.

So eventually we can have ideas like:

```kotlin
val operation = ...
```

where `operation` itself represents behavior.

This is the bridge from ordinary functions to:

- lambdas,
    
- higher-order functions,
    
- functional programming,
    
- collection operations,
    
- coroutine APIs.
    

We'll get there soon.

Don't worry about the syntax yet.

---

# 19. The difference between a function and its result

This is subtle and important.

Consider:

```kotlin
fun double(x: Int): Int {
    return x * 2
}
```

Here:

```kotlin
double
```

refers to the function.

But:

```kotlin
double(5)
```

means:

> Call the function with `5`.

The call produces:

```text
10
```

So don't confuse:

```text
function
```

with:

```text
result of calling the function
```

Similar to how:

```text
recipe
```

is not the same thing as:

```text
meal produced using the recipe
```

The function is the reusable definition of behavior.

The invocation is an execution of that behavior.

---

# 20. Function execution is temporary

Consider:

```kotlin
fun calculate() {
    val x = 10
    val y = 20

    println(x + y)
}
```

When `calculate()` is called, the function executes.

During that execution, `x` and `y` are available in its local scope.

When the function finishes, that particular invocation is over.

If you call it again:

```kotlin
calculate()
calculate()
```

there are two separate executions.

The local state from the first call isn't simply "the same call continuing."

Each invocation has its own local execution context.

We'll eventually connect this with the call stack and JVM execution, but for now this mental model is enough.

---

# 21. A useful question: "What does this function depend on?"

Consider:

```kotlin
fun calculateTotal(price: Double, quantity: Int): Double {
    return price * quantity
}
```

Its dependencies are obvious:

```text
price
quantity
```

Now consider:

```kotlin
var discountRate = 0.2

fun calculateTotal(price: Double): Double {
    return price * (1 - discountRate)
}
```

The function appears to take only `price`, but it secretly depends on:

```text
discountRate
```

from outside.

This makes the function's true dependency less obvious.

Later, you'll learn that there are legitimate reasons for functions and methods to depend on surrounding state. But **hidden dependencies increase complexity**, so we should learn to recognize them.

---

# 22. This leads to a powerful design principle

A good function often has:

```text
clear inputs
      ↓
clear behavior
      ↓
clear output
```

For example:

```kotlin
fun calculateTax(
    amount: Double,
    rate: Double
): Double {
    return amount * rate
}
```

You can understand this function almost entirely from its signature and body.

That's excellent.

Contrast it with something that silently reads five global variables and modifies two other objects.

The second function may still be necessary in an application, but it is harder to reason about.

---

# 23. Let's revisit our shopping program

Instead of:

```kotlin
fun main() {
    val price = 1000.0
    val quantity = 3
    val member = true

    // lots of calculations...
}
```

we can say:

```kotlin
fun calculateSubtotal(
    price: Double,
    quantity: Int
): Double {
    return price * quantity
}

fun calculateDiscount(
    subtotal: Double,
    isMember: Boolean
): Double {
    return if (isMember) {
        subtotal * 0.10
    } else {
        0.0
    }
}

fun calculateFinalPrice(
    subtotal: Double,
    discount: Double
): Double {
    return subtotal - discount
}
```

Then:

```kotlin
fun main() {
    val price = 1000.0
    val quantity = 3
    val isMember = true

    val subtotal = calculateSubtotal(price, quantity)
    val discount = calculateDiscount(subtotal, isMember)
    val finalPrice = calculateFinalPrice(subtotal, discount)

    println(finalPrice)
}
```

Now read `main()` almost like a workflow:

```text
calculate subtotal
       ↓
calculate discount
       ↓
calculate final price
       ↓
display it
```

The implementation details live elsewhere.

That's the beginning of good program structure.

---

# 24. A note about `return`

When Kotlin executes:

```kotlin
return result
```

two things happen conceptually:

1. The current function stops executing.
    
2. The value is handed back to the caller.
    

So in:

```kotlin
fun getNumber(): Int {
    println("Before")
    return 10
    println("After")
}
```

the `"After"` line can never execute.

The `return` has already ended the function.

This becomes particularly important when we use early returns for validation:

```kotlin
fun process(age: Int): String {
    if (age < 0) {
        return "Invalid age"
    }

    return "Age accepted"
}
```

We don't necessarily have to put everything into one giant nested `if`.

Early returns can sometimes make control flow clearer.

We'll learn when they help and when they make a function confusing.

---

# 25. One subtle point: `return` returns from the current function

At this stage, just remember:

```kotlin
fun outer() {
    inner()
}

fun inner(): Int {
    return 10
}
```

The `return` in `inner()` returns from `inner()`, not `outer()`.

Conceptually:

```text
outer
 ↓
inner
 ↓
return 10
 ↓
back to outer
```

This sounds obvious now, but later, with lambdas and inline functions, Kotlin's return behavior becomes considerably more interesting.

---

# 26. Your exercises

### Exercise 1 — Scope

Try to predict which variables are accessible:

```kotlin
fun main() {
    val a = 10

    if (a > 5) {
        val b = 20
        println(a)
        println(b)
    }

    println(a)
    println(b)
}
```

Before running it, identify exactly which line causes the problem and why.

---

### Exercise 2 — Inputs and outputs

Write:

```text
multiply(a, b)
```

that returns the product of two integers.

Then use the result in another calculation:

```text
multiply(5, 4) + 10
```

---

### Exercise 3 — Side effects

Write two functions:

```text
calculateSquare(number)
printSquare(number)
```

The first should return the square.

The second should print it.

Then explain to yourself why they have different responsibilities.

---

### Exercise 4 — Defaults

Create:

```text
greet(name, greeting)
```

where `greeting` defaults to `"Hello"`.

Make both of these work:

```kotlin
greet("Alice")
greet("Alice", "Good morning")
```

---

### Exercise 5 — Named arguments

Create:

```text
createProfile(name, age, city, isActive)
```

Give sensible defaults where appropriate.

Then call it using named arguments.

---

# 27. Challenge: separate calculation from side effects

Write a small program that calculates the final price of a product.

Your program should have functions for:

```text
calculateSubtotal
calculateTax
calculateFinalPrice
```

Try to make the calculation functions **return values rather than print them**.

Then let `main()` handle the final display.

For example, conceptually:

```text
Input
 ↓
calculation functions
 ↓
final value
 ↓
print
```

Don't worry about making it architecturally perfect.

The goal is to start noticing the difference between:

> **computing something**

and:

> **doing something externally visible**.

That distinction will become very valuable later.

---

# The bigger picture

Our understanding of functions is becoming much richer:

```text
Function
│
├── name
├── parameters
├── local scope
├── body
├── return type
├── returned value
└── side effects
```

And a function can be viewed from two perspectives.

### As a machine

```text
input
  ↓
computation
  ↓
output
```

### As an abstraction

```text
complex implementation
        ↓
     function
        ↓
simple named concept
```

Both views are useful.

---

There's one final idea I want you to notice before we move on.

We've been writing functions like:

```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}
```

The function itself has a name, parameters, a type, and behavior.

But what if **a function could itself be stored in a variable?**

What if we could do something conceptually like:

```text
put this behavior into a variable
      ↓
pass it to another function
      ↓
call it later
```

That is where Kotlin starts becoming much more expressive.

And that's our next major topic:

# Functions as values, function types, and lambdas

This is the point where you'll start understanding code such as:

```kotlin
numbers.map { it * 2 }
```

instead of treating it as some strange Kotlin incantation.