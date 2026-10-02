

Now we're getting into the first concept that makes a program feel like an actual program.

So far we've mostly done things like:

```kotlin
println(10 + 20)
```

But imagine you're building an expense calculator.

You have:

```text
price = 250
quantity = 3
```

You calculate:

```text
total = price × quantity
```

Then perhaps you calculate:

```text
tax = total × 0.18
```

and finally:

```text
finalAmount = total + tax
```

A program needs some way to **keep track of these pieces of information**.

That's what variables are for.

---

## 1. The simplest variable

In Kotlin:

```kotlin
val age = 25
```

This is a variable declaration.

You have created a name:

```text
age
```

and associated it with the value:

```text
25
```

You can then use that name elsewhere:

```kotlin
fun main() {
    val age = 25

    println(age)
    println(age + 5)
}
```

Output:

```text
25
30
```

Notice what happened.

When Kotlin sees:

```kotlin
println(age)
```

it doesn't print the word `age`.

It looks up what `age` represents and uses that value:

```text
age
 ↓
25
```

So you can think of a variable as giving a piece of data a **name that we can refer to later**.

---

# 2. Why do we need variables?

Compare these two programs.

### Without variables

```kotlin
fun main() {
    println(250 * 3)
    println((250 * 3) * 0.18)
    println((250 * 3) + ((250 * 3) * 0.18))
}
```

This works, but it's becoming ugly very quickly.

Now look at:

```kotlin
fun main() {
    val price = 250
    val quantity = 3

    val total = price * quantity
    val tax = total * 0.18

    val finalAmount = total + tax

    println(finalAmount)
}
```

Much easier to understand.

The second program has names that describe what the values mean.

That's one of the most important purposes of variables:

> **Variables let us give meaningful names to information so that our programs can work with that information later.**

---

# 3. `val` vs `var`

This is one of the first things you absolutely need to understand in Kotlin.

Kotlin gives us two common ways to declare a variable:

```kotlin
val
var
```

The basic distinction is:

### `val`

```kotlin
val age = 25
```

Once `age` has been assigned, you cannot assign a new value to `age`.

### `var`

```kotlin
var age = 25
age = 26
```

A `var` can be reassigned.

So:

```kotlin
val age = 25
age = 26
```

will produce a compilation error.

But:

```kotlin
var age = 25
age = 26
```

is perfectly valid.

---

# 4. Think of `val` as "this name cannot be reassigned"

This is an important wording difference.

Don't initially think:

> "`val` means the value can never change."

That's not quite correct.

A better mental model is:

> **`val` means this variable/reference cannot be reassigned to another value.**

For example:

```kotlin
val x = 10
```

You cannot later do:

```kotlin
x = 20
```

But this doesn't mean every object referred to by a `val` is necessarily immutable.

We'll come back to that distinction later when we study objects and collections.

For now:

```text
val  → cannot reassign
var  → can reassign
```

---

# 5. Why does Kotlin have `val`?

Here's a design decision hiding inside the language.

Suppose I write:

```kotlin
val birthYear = 2000
```

That tells someone reading the code:

> This name isn't going to be assigned something else later.

That makes the code easier to reason about.

Compare:

```kotlin
var birthYear = 2000
```

A reader now has to wonder:

> Is this going to change somewhere?

Maybe it will. Maybe it won't.

Using `val` communicates an intention.

That's why you'll often see Kotlin code with many more `val`s than `var`s.

A useful rule for us will be:

> **Use `val` when you don't need reassignment. Use `var` when you genuinely do.**

Don't treat this as a law. There are legitimate uses for `var`.

---

# 6. A variable doesn't have to be used immediately

You can write:

```kotlin
fun main() {
    val name = "Alice"
    val age = 25
    val city = "Pune"

    println(name)
    println(age)
    println(city)
}
```

Each declaration creates a named piece of data that we can use later.

The order matters.

This works:

```kotlin
val age = 25
println(age)
```

But this does not:

```kotlin
println(age)
val age = 25
```

Why?

Because when Kotlin reaches the first line, `age` hasn't been declared in that scope yet.

This introduces an important idea:

> **Names exist within particular regions of code.**

We'll study scope more deeply later.

---

# 7. Reassignment

With `var`, you can change what the variable refers to.

```kotlin
var score = 10

score = 20
score = 30

println(score)
```

Output:

```text
30
```

The important thing is that:

```kotlin
score = 20
```

is not creating another variable called `score`.

You're **reassigning the existing `score` variable**.

Conceptually:

```text
Initially:

score ──→ 10

Then:

score ──→ 20

Then:

score ──→ 30
```

So after the final assignment, reading `score` gives `30`.

---

# 8. `=` does NOT mean "is equal to"

This is a very common beginner mistake.

Consider:

```kotlin
var age = 20
age = 21
```

People coming from mathematics sometimes instinctively read:

```text
age = 21
```

as:

> "age is equal to 21."

That's not quite what the programming statement is doing.

Here `=` is primarily an **assignment operator**.

It's telling the program:

> Put this value into this variable.

We'll later encounter `==`, which is used for comparison.

So keep these separate:

```text
=   → assignment

==  → equality comparison
```

We'll study `==` properly when we get to conditions.

---

# 9. A useful example

Suppose we have a bank account.

```kotlin
var balance = 1000
```

Then:

```kotlin
balance = balance + 500
```

At first this may look strange.

Read it carefully.

The right-hand side:

```kotlin
balance + 500
```

uses the **current** value of `balance`.

So:

```text
balance = 1000

balance + 500
        ↓
1500
```

Then the assignment happens:

```text
balance = 1500
```

So the whole operation is conceptually:

```text
old balance
     ↓
read 1000
     ↓
add 500
     ↓
get 1500
     ↓
store 1500 back into balance
```

Therefore:

```kotlin
fun main() {
    var balance = 1000

    balance = balance + 500

    println(balance)
}
```

prints:

```text
1500
```

This pattern is fundamental to programming.

---

# 10. Shorthand reassignment

Because this pattern is so common:

```kotlin
balance = balance + 500
```

Kotlin lets us write:

```kotlin
balance += 500
```

Similarly:

```kotlin
balance -= 100
balance *= 2
balance /= 5
```

For example:

```kotlin
var score = 10

score += 5
println(score)
```

prints:

```text
15
```

We'll use these frequently.

---

# 11. What does Kotlin know about the type?

Look at:

```kotlin
val age = 25
```

We didn't explicitly say:

```kotlin
Int
```

Yet Kotlin knows `age` is an integer type.

This is called **type inference**.

Kotlin looks at:

```kotlin
25
```

and can infer an appropriate type.

So conceptually:

```kotlin
val age = 25
```

is understood approximately as:

```kotlin
val age: Int = 25
```

The second form explicitly states the type.

---

# 12. You can specify the type yourself

You can write:

```kotlin
val age: Int = 25
```

The syntax is:

```text
name : Type
```

So:

```kotlin
val age: Int = 25
```

means:

```text
name → age
type → Int
value → 25
```

This is your first encounter with a major Kotlin idea:

> **Every value has a type.**

We'll eventually go very deep into Kotlin's type system.

For now, just recognize the basic syntax.

---

# 13. More examples of explicit types

```kotlin
val age: Int = 25
val price: Double = 19.99
val name: String = "Alice"
val isLoggedIn: Boolean = true
val initial: Char = 'A'
```

We'll learn all of these types properly.

For now, notice the pattern:

```kotlin
val variableName: Type = value
```

For example:

```kotlin
val name: String = "Alice"
```

---

# 14. Why not specify the type everywhere?

You might wonder:

> "If Kotlin can infer the type, why would I ever write it?"

Sometimes inference makes code cleaner:

```kotlin
val age = 25
```

There's no useful information gained by writing:

```kotlin
val age: Int = 25
```

because `25` already makes the type obvious.

But explicit types can become useful when:

- the intended type isn't obvious
    
- you're defining an API
    
- you're working with function signatures
    
- you need a broader/different type
    
- inference can't determine what you mean
    

We'll learn these cases naturally rather than memorizing a rule.

---

# 15. Here's something subtle

Consider:

```kotlin
val x = 10
val y = 20

val total = x + y
```

What is the type of `total`?

Kotlin evaluates the expression:

```text
x + y
```

which produces:

```text
30
```

and that result has a type.

So type inference isn't only happening for literal values.

Kotlin can infer types based on expressions as well.

This is one reason you'll often see very little explicit type annotation in idiomatic Kotlin code.

---

# 16. Values vs variables

This distinction is worth slowing down for.

Take:

```kotlin
val age = 25
```

There are two related but different ideas:

```text
age
```

is a **name** we can use.

```text
25
```

is a **value**.

The variable gives us a way to refer to that value.

So when you write:

```kotlin
println(age)
```

Kotlin essentially follows the name and obtains the associated value.

Conceptually:

```text
       age
        │
        ▼
       25
        │
        ▼
    println
```

Don't get too attached to this as a literal description of the JVM's memory implementation. It's a conceptual model that helps us reason about code.

Later, when we study references, objects and the JVM, we'll refine the model.

---

# 17. `val` does not automatically mean immutable

This is one of the most important subtleties in Kotlin.

Consider:

```kotlin
val numbers = mutableListOf(1, 2, 3)
```

Can we do:

```kotlin
numbers = mutableListOf(4, 5, 6)
```

No.

Because `numbers` is a `val`.

But can the list itself change?

Yes:

```kotlin
numbers.add(4)
```

Now it contains:

```text
[1, 2, 3, 4]
```

So:

```text
val
```

does **not** necessarily mean:

```text
immutable object
```

It means the variable can't be reassigned.

This is a distinction we'll revisit many times.

For the moment:

```text
val numbers = ...
```

means:

> `numbers` can't be made to refer to a different object.

It does not necessarily mean:

> The object referred to by `numbers` can never change.

Don't worry if that feels slightly abstract right now. Once we study objects and collections, it'll become much clearer.

---

# 18. Naming variables properly

You can technically choose many names, but good names matter.

Compare:

```kotlin
val x = 250
val y = 3
val z = x * y
```

with:

```kotlin
val price = 250
val quantity = 3
val total = price * quantity
```

Both might execute identically.

But the second tells us what the program means.

Good programming isn't just about making the computer understand.

It's also about making **humans** understand.

Kotlin conventionally uses **camelCase** for variables and functions:

```kotlin
val firstName = "Alice"
val accountBalance = 5000
val numberOfStudents = 30
```

Not:

```kotlin
val first_name = "Alice"
```

and usually not:

```kotlin
val FirstName = "Alice"
```

We'll cover naming conventions in more detail later.

---

# 19. A little program to study

Look at this carefully:

```kotlin
fun main() {
    val price = 100
    val quantity = 3

    var discount = 0

    val subtotal = price * quantity

    discount = 20

    val finalPrice = subtotal - discount

    println(finalPrice)
}
```

Let's mentally execute it.

First:

```text
price = 100
quantity = 3
discount = 0
```

Then:

```text
subtotal = 100 × 3
         = 300
```

Then:

```text
discount = 20
```

Then:

```text
finalPrice = 300 - 20
           = 280
```

So the output is:

```text
280
```

Notice that `discount` was declared as `var` because we assigned a different value to it later.

But these:

```kotlin
val price = 100
val quantity = 3
val subtotal = price * quantity
val finalPrice = subtotal - discount
```

are never reassigned, so `val` is appropriate.

---

# 20. A question I want you to start asking

Whenever you see a variable declaration, ask:

> **Does this value actually need to change?**

For example:

```kotlin
val birthYear = 2000
```

Why would we make this:

```kotlin
var birthYear = 2000
```

if we're never going to change it?

There's no benefit.

`val` gives us a useful guarantee:

```text
birthYear won't be reassigned.
```

That makes reasoning about the program easier.

---

# 21. But don't turn this into "var is bad"

This is another common oversimplification.

`var` is not bad.

Sometimes the state genuinely changes.

For example:

```kotlin
var currentPage = 1

currentPage += 1
```

That's completely reasonable.

Or:

```kotlin
var temperature = 25

temperature = 27
```

If the concept itself represents changing state, `var` may be exactly what we need.

The principle is:

> **Don't use `var` merely because you can. Use it when reassignment is part of the logic.**

---

# 22. Your first mini thought experiment

Without running anything, what do you think this prints?

```kotlin
fun main() {
    var x = 10

    x = x + 5
    x = x * 2

    println(x)
}
```

Trace it manually:

```text
x = 10

x = x + 5
  = 10 + 5
  = 15

x = x * 2
  = 15 * 2
  = 30
```

Therefore:

```text
30
```

This sort of tracing is something I want you to become very comfortable with.

When debugging later, you'll often mentally execute code in exactly this way.

---

# 23. One more thought experiment

What about:

```kotlin
fun main() {
    val x = 10

    x = 20

    println(x)
}
```

This doesn't run.

Why?

Because `x` was declared with `val`, and you're attempting to reassign it.

The issue isn't the value `20`.

The issue is that:

```text
x
```

is read-only with respect to reassignment.

Change it to:

```kotlin
var x = 10
```

and it becomes valid.

---

# 24. Another subtle point: declaration vs assignment

These two lines are doing different things:

```kotlin
val age = 25
```

and:

```kotlin
age = 26
```

The first is a **declaration**.

You're introducing the name `age`.

The second is an **assignment**.

You're assigning a new value to an existing variable.

We'll use these words precisely throughout the course because they become important later.

---

# 25. What you've learned so far

You should now have a mental model roughly like this:

```text
                    Variable
                       │
              ┌────────┴────────┐
              │                 │
             val               var
              │                 │
       can't reassign      can reassign
```

And:

```text
val age = 25
   │      │
   │      └── value
   │
   └── name
```

You can also explicitly specify the type:

```kotlin
val age: Int = 25
```

while Kotlin can often infer it:

```kotlin
val age = 25
```

And remember the important subtlety:

```text
val ≠ necessarily immutable object
```

It means **the variable/reference can't be reassigned**.

---

# Let's make this practical

Create a new Kotlin file and write this yourself rather than copying it blindly:

```kotlin
fun main() {
    val name = "Your Name"
    var age = 20

    println(name)
    println(age)

    age = age + 1

    println(age)
}
```

Replace `"Your Name"` with your name.

Before running it, predict exactly what it will print.

Then make these changes:

### Experiment 1

Change:

```kotlin
var age = 20
```

to:

```kotlin
val age = 20
```

and leave:

```kotlin
age = age + 1
```

What does IntelliJ complain about?

Don't just fix it. Understand **why**.

### Experiment 2

Create:

```kotlin
val price = 250
val quantity = 4
```

Then calculate the total using another variable.

### Experiment 3

Create a `var` called `score`.

Start it at `0`, then increase it three times.

Try both:

```kotlin
score = score + 10
```

and:

```kotlin
score += 10
```

and observe that they accomplish the same basic reassignment.

---

## One challenge

Don't look at the answer immediately. Try to reason through this:

```kotlin
fun main() {
    var a = 10
    val b = 20

    a = a + b
    a = a * 2

    println(a)
    println(b)
}
```

Work out the two outputs **before running the program**.

Then, after you've run it, ask yourself:

> Why did `a` change but `b` didn't?

That question is more important than the output itself.

---

The next lesson will build naturally from here: **types and numbers**.

We'll investigate why Kotlin needs things like `Int`, `Long`, `Double`, and `Float`, why `10 / 3` can surprise you, what exactly happens when you mix different numeric types, and why Kotlin doesn't silently convert everything the way some languages do.