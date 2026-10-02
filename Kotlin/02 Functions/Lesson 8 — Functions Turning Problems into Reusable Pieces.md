
# Lesson 8 — Functions: Turning Problems into Reusable Pieces

Until now, most of our programs have looked like this:

```kotlin
fun main() {
    val price = 100
    val quantity = 3

    val total = price * quantity

    println(total)
}
```

This is fine while our program is tiny.

But imagine `main()` becoming 500 lines long.

You might have code for:

- calculating prices
    
- validating users
    
- calculating taxes
    
- printing reports
    
- processing orders
    
- checking permissions
    
- formatting messages
    

Everything would be mixed together.

That's where **functions** become fundamental.

A function lets us take some piece of behavior, give it a name, and use it whenever we need it.

---

# 1. What is a function?

At a high level:

> **A function is a named piece of behavior that can receive input, perform some computation, and optionally produce an output.**

Think about a mathematical function:

```text
f(x) = x × 2
```

You give it:

```text
5
```

and it gives you:

```text
10
```

Programming functions follow a very similar idea.

For example:

```kotlin
fun double(number: Int): Int {
    return number * 2
}
```

Then:

```kotlin
val result = double(5)
```

gives:

```text
result = 10
```

We now have a reusable operation called `double`.

---

# 2. The simplest function

We have actually been using one since Lesson 1:

```kotlin
fun main() {
    println("Hello")
}
```

Let's create our own:

```kotlin
fun sayHello() {
    println("Hello!")
}
```

Then call it:

```kotlin
fun main() {
    sayHello()
}
```

Output:

```text
Hello!
```

So there are two separate things happening:

### Declaring the function

```kotlin
fun sayHello() {
    println("Hello!")
}
```

We're defining what `sayHello` means.

### Calling the function

```kotlin
sayHello()
```

We're saying:

> Execute that function now.

That distinction is important.

---

# 3. Why bother with a function?

We could have simply written:

```kotlin
fun main() {
    println("Hello!")
}
```

So why create:

```kotlin
fun sayHello() {
    println("Hello!")
}
```

?

Because now the behavior has a **name**.

We can use it multiple times:

```kotlin
fun main() {
    sayHello()
    sayHello()
    sayHello()
}
```

Output:

```text
Hello!
Hello!
Hello!
```

Instead of repeating the implementation:

```kotlin
println("Hello!")
println("Hello!")
println("Hello!")
```

We've given a concept a name.

That's one of the deepest purposes of functions:

> **Functions allow us to name pieces of logic.**

---

# 4. Functions are about abstraction

Suppose I write:

```kotlin
printInvoice()
```

A person reading this doesn't need to know immediately how the invoice is printed.

The name tells them:

> "This part of the program prints an invoice."

Inside the function there might be:

```kotlin
fun printInvoice() {
    calculateSubtotal()
    calculateTax()
    formatCustomerDetails()
    println(...)
}
```

And those functions might themselves call others.

So instead of reasoning about hundreds of lines simultaneously, we can reason about the program at several levels:

```text
main
 ↓
processOrder
 ↓
calculateTotal
 ↓
calculateTax
```

This ability to hide implementation details behind meaningful names is called **abstraction**.

We'll spend a lot of time on abstraction later because it is one of the foundations of software engineering.

---

# 5. A function can receive input

Our `sayHello()` function always says the same thing.

That's not very flexible.

What if we want to greet different people?

We can give the function a **parameter**.

```kotlin
fun greet(name: String) {
    println("Hello, $name!")
}
```

Now:

```kotlin
fun main() {
    greet("Alice")
    greet("Bob")
    greet("Charlie")
}
```

Output:

```text
Hello, Alice!
Hello, Bob!
Hello, Charlie!
```

The function has a parameter:

```kotlin
name: String
```

and each call supplies an argument:

```kotlin
greet("Alice")
```

---

# 6. Parameter vs argument

These terms are often used interchangeably in casual conversation, but technically they're different.

In:

```kotlin
fun greet(name: String) {
    println("Hello, $name!")
}
```

`name` is a **parameter**.

It's part of the function definition.

Then:

```kotlin
greet("Alice")
```

contains `"Alice"`, which is the **argument** supplied to the function call.

So:

```text
Function definition:
    parameter

Function call:
    argument
```

For example:

```kotlin
fun greet(name: String) { ... }
             ↑
          parameter

greet("Alice")
      ↑
    argument
```

We'll use the terms carefully as we go.

---

# 7. What happens when the function is called?

Take:

```kotlin
fun greet(name: String) {
    println("Hello, $name!")
}

fun main() {
    greet("Alice")
}
```

Conceptually:

```text
main
 ↓
call greet("Alice")
 ↓
name receives "Alice"
 ↓
execute function body
 ↓
println("Hello, Alice!")
```

You can think of the parameter as a local name created for the duration of that function call.

So inside:

```kotlin
fun greet(name: String) {
    println(name)
}
```

`name` refers to the argument supplied to that particular invocation.

---

# 8. Functions can take multiple parameters

For example:

```kotlin
fun add(a: Int, b: Int) {
    println(a + b)
}
```

Then:

```kotlin
fun main() {
    add(10, 20)
    add(5, 7)
}
```

Output:

```text
30
12
```

Here:

```text
a → first argument
b → second argument
```

So:

```kotlin
add(10, 20)
```

conceptually creates:

```text
a = 10
b = 20
```

and then runs:

```kotlin
println(a + b)
```

---

# 9. But printing isn't the most useful thing a function can do

Consider:

```kotlin
fun add(a: Int, b: Int) {
    println(a + b)
}
```

What if we want to use the answer somewhere else?

Suppose:

```kotlin
val x = add(10, 20)
```

What should `x` contain?

Our function currently doesn't return a useful value.

This is where **return values** come in.

---

# 10. Returning a value

We can write:

```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}
```

Now:

```kotlin
val result = add(10, 20)
println(result)
```

prints:

```text
30
```

Let's examine the function declaration:

```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}
```

We now have:

```text
fun
 ↓
function declaration

add
 ↓
function name

(a: Int, b: Int)
 ↓
parameters

: Int
 ↓
return type

return a + b
 ↓
value returned to caller
```

This is a major step forward.

---

# 11. Think of a function as a machine

This is a useful mental model:

```text
         input
           ↓
    ┌─────────────┐
    │   function  │
    │             │
    │ computation │
    │             │
    └─────────────┘
           ↓
         output
```

For:

```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}
```

we have:

```text
10 ──┐
     ├──→ add ──→ 30
20 ──┘
```

For:

```kotlin
fun double(number: Int): Int {
    return number * 2
}
```

we have:

```text
5
↓
double
↓
10
```

That mental model will take us a long way.

---

# 12. `return` gives control back to the caller

Consider:

```kotlin
fun add(a: Int, b: Int): Int {
    val result = a + b
    return result
}
```

When Kotlin reaches:

```kotlin
return result
```

the function finishes and sends that value back to wherever the function was called.

So:

```kotlin
val x = add(10, 20)
```

eventually becomes conceptually:

```text
add(10, 20)
   ↓
30
   ↓
x = 30
```

This is exactly what we mean when we say the function **returns** a value.

---

# 13. A function can use its return value inside a larger expression

Because a function call can produce a value, we can do:

```kotlin
val total = add(10, 20) * 2
```

First:

```text
add(10, 20)
    ↓
30
```

Then:

```text
30 * 2
   ↓
60
```

So:

```text
total = 60
```

This connects directly to our earlier lesson about expressions.

Remember:

> An expression produces a value.

A function call can itself be an expression when the function returns a value.

For example:

```kotlin
add(10, 20)
```

produces:

```text
30
```

---

# 14. Functions can call other functions

This is where programs start becoming hierarchical.

```kotlin
fun double(number: Int): Int {
    return number * 2
}

fun quadruple(number: Int): Int {
    return double(double(number))
}
```

Now:

```kotlin
val result = quadruple(5)
```

Trace it:

```text
quadruple(5)
    ↓
double(double(5))
    ↓
double(10)
    ↓
20
```

This may initially feel a little abstract.

But larger programs are full of exactly this sort of composition.

A high-level function uses smaller functions to accomplish its job.

---

# 15. Function composition

Suppose we have:

```kotlin
fun double(x: Int): Int {
    return x * 2
}

fun addTen(x: Int): Int {
    return x + 10
}
```

We can combine them:

```kotlin
val result = addTen(double(5))
```

Work from the inside out:

```text
double(5)
  ↓
10

addTen(10)
  ↓
20
```

So:

```text
result = 20
```

You are beginning to see the idea of building larger computations from smaller operations.

This becomes enormously important later when we learn higher-order functions and functional programming.

---

# 16. Functions don't have to return `Int`

They can return any appropriate type.

For example:

```kotlin
fun getName(): String {
    return "Alice"
}
```

Or:

```kotlin
fun isAdult(age: Int): Boolean {
    return age >= 18
}
```

Then:

```kotlin
val name = getName()
val adult = isAdult(25)
```

So functions can produce:

```text
Int
String
Boolean
Double
objects
collections
other functions
...
```

We'll eventually see that a function can be treated like a value itself.

---

# 17. Functions that don't return a meaningful value

What about:

```kotlin
fun greet(name: String) {
    println("Hello, $name!")
}
```

There's no explicit return type.

Kotlin treats this as returning:

```text
Unit
```

So conceptually:

```kotlin
fun greet(name: String): Unit {
    println("Hello, $name!")
}
```

is the same basic idea.

`Unit` means:

> The function completes, but doesn't produce a meaningful result value for the caller.

You don't usually need to write `: Unit` explicitly.

This is why:

```kotlin
fun greet(name: String) {
    println("Hello, $name!")
}
```

is idiomatic Kotlin.

---

# 18. The difference between "printing" and "returning"

This is very important.

Consider:

```kotlin
fun add(a: Int, b: Int) {
    println(a + b)
}
```

versus:

```kotlin
fun add(a: Int, b: Int): Int {
    return a + b
}
```

The first function **prints** the result.

The second **returns** the result.

Why does that matter?

Because returned values can be reused:

```kotlin
val result = add(10, 20)
```

With the first version, that's not what the function is designed to do.

With the second:

```text
add(10, 20)
      ↓
    value
      ↓
use it somewhere else
```

This is a crucial distinction.

A function that returns data is generally much more reusable than one that immediately prints it.

Printing is an effect.

Returning is providing data to the caller.

That distinction will become very important later when we discuss software design and testing.

---

# 19. A real example

Let's revisit our earlier shopping example.

Instead of putting everything into `main()`:

```kotlin
fun main() {
    val price = 100
    val quantity = 3
    val subtotal = price * quantity
    val tax = subtotal * 0.18
    val total = subtotal + tax

    println(total)
}
```

we could introduce functions:

```kotlin
fun calculateSubtotal(price: Double, quantity: Int): Double {
    return price * quantity
}

fun calculateTax(subtotal: Double): Double {
    return subtotal * 0.18
}
```

Then:

```kotlin
fun main() {
    val price = 100.0
    val quantity = 3

    val subtotal = calculateSubtotal(price, quantity)
    val tax = calculateTax(subtotal)
    val total = subtotal + tax

    println(total)
}
```

Now `main()` describes the overall process:

```text
calculate subtotal
      ↓
calculate tax
      ↓
calculate total
      ↓
print
```

The detailed implementation has been moved into smaller named functions.

That's abstraction in practice.

---

# 20. Functions can have default arguments

Suppose:

```kotlin
fun greet(name: String, punctuation: String = "!") {
    println("Hello, $name$punctuation")
}
```

Now you can write:

```kotlin
greet("Alice")
```

and Kotlin uses:

```text
punctuation = "!"
```

Or:

```kotlin
greet("Alice", "!!!")
```

which produces:

```text
Hello, Alice!!!
```

Default arguments let functions have flexible APIs without requiring multiple versions of the function.

We'll study them more carefully when we discuss function design.

---

# 21. Named arguments

Suppose:

```kotlin
fun createUser(name: String, age: Int, active: Boolean) {
    ...
}
```

You could call:

```kotlin
createUser("Alice", 25, true)
```

But you can also use named arguments:

```kotlin
createUser(
    name = "Alice",
    age = 25,
    active = true
)
```

This can make calls much easier to understand, especially when multiple arguments have similar types.

For example, imagine:

```kotlin
sendMoney(100, 200, 500)
```

What do those numbers mean?

A named call might make the intention clearer:

```kotlin
sendMoney(
    fromAccount = 100,
    toAccount = 200,
    amount = 500
)
```

The names become part of the readability of the API.

---

# 22. Function bodies can sometimes be shorter

This:

```kotlin
fun double(number: Int): Int {
    return number * 2
}
```

can be written:

```kotlin
fun double(number: Int): Int = number * 2
```

This is called an **expression body**.

Because:

```kotlin
number * 2
```

is an expression that produces the returned value.

We'll use this form sometimes, but not blindly.

For a tiny function:

```kotlin
fun square(x: Int) = x * x
```

is nice.

For complicated logic, a normal block can be clearer.

Again, our goal is clarity, not maximum compactness.

---

# 23. Local variables inside functions

Consider:

```kotlin
fun calculateTotal(price: Double, quantity: Int): Double {
    val subtotal = price * quantity
    val tax = subtotal * 0.18

    return subtotal + tax
}
```

`subtotal` and `tax` are local variables.

They exist inside the function.

Another function cannot simply use them:

```kotlin
fun anotherFunction() {
    println(subtotal) // not available here
}
```

This introduces **scope**.

A name exists within a particular region of code.

We'll eventually study scope carefully because it affects:

- variables
    
- functions
    
- classes
    
- loops
    
- lambdas
    
- visibility
    

For now, just notice that local variables belong to the function in which they're declared.

---

# 24. A function can have multiple `return` paths

For example:

```kotlin
fun absoluteValue(number: Int): Int {
    if (number >= 0) {
        return number
    }

    return -number
}
```

If:

```text
number = 5
```

we return `5`.

If:

```text
number = -5
```

the first condition is false, and we return:

```text
5
```

So functions can have different execution paths that eventually return a value.

This connects functions with the control-flow concepts we learned earlier.

---

# 25. Return values and `if`

Remember that `if` itself can produce a value.

That means we can write:

```kotlin
fun absoluteValue(number: Int): Int {
    return if (number >= 0) {
        number
    } else {
        -number
    }
}
```

This is a nice example of Kotlin's expression-oriented design.

The `if` produces either:

```text
number
```

or:

```text
-number
```

and that result gets returned.

---

# 26. The function contract

There's a useful way to think about a function:

```text
Function name
      +
Inputs
      +
Output
      +
Behavior
```

For:

```kotlin
fun isAdult(age: Int): Boolean {
    return age >= 18
}
```

the contract is approximately:

```text
Name:
isAdult

Input:
an Int representing age

Output:
Boolean

Behavior:
true when age is at least 18
```

A well-designed function often has a clear contract.

This becomes extremely important in larger codebases because other parts of the program depend on that contract.

---

# 27. Functions let you name concepts

This is perhaps even more important than reuse.

Compare:

```kotlin
if (age >= 18 && hasId && !isBanned) {
    ...
}
```

with:

```kotlin
if (canEnter(age, hasId, isBanned)) {
    ...
}
```

where:

```kotlin
fun canEnter(age: Int, hasId: Boolean, isBanned: Boolean): Boolean {
    return age >= 18 && hasId && !isBanned
}
```

The second version gives a name to a concept:

```text
canEnter
```

Now other programmers can reason about the code at a higher level.

This is one of the central ideas of abstraction:

> **A good function name can communicate intent.**

---

# 28. But don't create functions for every single line

You could write:

```kotlin
fun addTwoNumbers(a: Int, b: Int): Int {
    return a + b
}
```

That's fine when the concept itself is meaningful.

But don't imagine that every two lines of code must become a function.

Functions are useful when they provide things such as:

- meaningful abstraction
    
- reuse
    
- isolation
    
- testability
    
- clearer organization
    

For example:

```kotlin
val total = price + tax
```

doesn't automatically need:

```kotlin
fun addPriceAndTax(...)
```

The function should represent a useful concept, not merely exist because functions are "good."

---

# 29. Let's look at a bigger example

Suppose:

```kotlin
fun calculateDiscount(price: Double, isMember: Boolean): Double {
    return if (isMember) {
        price * 0.10
    } else {
        0.0
    }
}

fun calculateFinalPrice(price: Double, isMember: Boolean): Double {
    val discount = calculateDiscount(price, isMember)
    return price - discount
}
```

Then:

```kotlin
fun main() {
    val price = 1000.0
    val isMember = true

    val finalPrice = calculateFinalPrice(price, isMember)

    println(finalPrice)
}
```

Now look at `main()`.

It says, essentially:

```text
Take price
 ↓
Calculate final price
 ↓
Print it
```

It doesn't contain all the details.

And that's a good sign.

As programs grow, we want each level to be understandable without requiring us to hold the entire implementation in our heads at once.

---

# 30. One subtle idea: functions don't automatically "remember" previous calls

Consider:

```kotlin
fun double(x: Int): Int {
    return x * 2
}
```

Calling:

```kotlin
double(5)
```

doesn't permanently change `double`.

You can then call:

```kotlin
double(20)
```

and it independently calculates:

```text
40
```

Each invocation is its own execution.

This becomes much more interesting once functions have mutable state, objects, closures and recursion, but for now think of a simple function as:

```text
input → computation → output
```

---

# 31. Your exercises

Don't just type these. Try to design the functions yourself.

### Exercise 1

Write:

```text
square(number)
```

which returns the square of a number.

For example:

```text
square(5) → 25
square(10) → 100
```

---

### Exercise 2

Write:

```text
isEven(number)
```

which returns `true` when the number is even and `false` otherwise.

Remember what we learned about `%`.

---

### Exercise 3

Write:

```text
max(a, b)
```

which returns the larger of two integers.

Try doing it with an `if` expression.

---

### Exercise 4

Write:

```text
calculateDiscount(price, isMember)
```

with these rules:

```text
member     → 10% discount
non-member → 0%
```

The function should **return the discount**, not print it.

Then use the returned value to calculate a final price.

---

### Exercise 5

Write:

```text
calculateAverage(a, b, c)
```

which returns the average of three numbers.

Be careful about integer division.

Ask yourself:

> What types should the parameters be if I want an accurate decimal average?

---

# 32. One challenge

Create these three functions:

```text
calculateSubtotal
calculateDiscount
calculateFinalPrice
```

Then make `main()` use them to calculate a purchase.

The goal isn't merely to get the correct final number.

Try to make `main()` read almost like a description of the problem.

Something like:

```text
get purchase information
       ↓
calculate subtotal
       ↓
calculate discount
       ↓
calculate final price
       ↓
display result
```

You're beginning to practice **decomposition**: taking one larger problem and splitting it into smaller meaningful operations.

---

# One final mental model

We've now added another major piece to our toolbox:

```text
Values
   ↓
Expressions
   ↓
Decisions
   ↓
Loops
   ↓
Functions
```

And functions allow us to turn this:

```text
one giant program
```

into:

```text
main
 ├── calculateSubtotal()
 ├── calculateDiscount()
 ├── calculateFinalPrice()
 └── printReceipt()
```

Then those functions can themselves be broken down further.

That's how large programs become manageable.

The next lesson is going to be especially important because we're going to **deepen our understanding of functions** rather than immediately jumping to another unrelated topic.

We'll look at **parameters, return values, scope, local functions, default/named arguments, and the distinction between a function's input, output, and side effects**. Then we'll move into one of Kotlin's most powerful features: **functions as values and lambdas**.