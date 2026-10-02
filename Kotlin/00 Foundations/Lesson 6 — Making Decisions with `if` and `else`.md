
# Lesson 6 — Making Decisions with `if` and `else`

So far, our programs have been **straight-line programs**.

For example:

```kotlin
fun main() {
    val age = 20

    println("Hello")
    println(age)
    println("Goodbye")
}
```

The computer essentially does:

```text
1. Print Hello
2. Print age
3. Print Goodbye
```

Every time you run it, the same sequence happens.

But real programs can't work like that.

A program needs to be able to say:

> "Do this **if** something is true."

For example:

```text
If the user is logged in → show dashboard
Otherwise → show login screen
```

or:

```text
If balance is sufficient → allow withdrawal
Otherwise → reject withdrawal
```

This is called **control flow**.

And today we're going to learn the first major control-flow construct:

# `if`

---

## 1. The basic idea

Suppose:

```kotlin
val age = 20
```

We want:

> If the person is 18 or older, print `"Adult"`.

Kotlin:

```kotlin
if (age >= 18) {
    println("Adult")
}
```

Read that almost like English:

```text
if age is greater than or equal to 18
    then execute this code
```

The condition:

```kotlin
age >= 18
```

is an expression that produces a `Boolean`.

Since:

```text
20 >= 18
```

is:

```text
true
```

the code inside the braces runs.

---

# 2. The anatomy of an `if`

Look at:

```kotlin
if (age >= 18) {
    println("Adult")
}
```

There are three important parts.

### `if`

Tells Kotlin:

> We're making a decision.

### `(age >= 18)`

This is the **condition**.

It must result in a Boolean.

### `{ ... }`

This is the **body**.

These are the instructions Kotlin executes when the condition is true.

So conceptually:

```text
             if
              ↓
       evaluate condition
              ↓
         true / false
          /        \
       true       false
        ↓            ↓
     execute      skip
      body         body
```

This is the first time our program can actually **choose whether something happens**.

---

# 3. What if the condition is false?

Suppose:

```kotlin
val age = 15

if (age >= 18) {
    println("Adult")
}
```

The condition becomes:

```text
15 >= 18
   ↓
false
```

So Kotlin simply skips the body.

Nothing is printed.

This is important:

> An `if` statement does not mean "the body will run."

It means:

> "Evaluate this condition. Run the body **only if** it is true."

---

# 4. Add `else`

Usually we don't want to simply do nothing when the condition is false.

We want an alternative.

For example:

```kotlin
val age = 15

if (age >= 18) {
    println("Adult")
} else {
    println("Minor")
}
```

Now there are two possible paths:

```text
              age >= 18?
               /     \
            true     false
             ↓         ↓
          Adult      Minor
```

Exactly one branch executes.

For `age = 15`:

```text
15 >= 18
   ↓
false
   ↓
else
   ↓
"Minor"
```

For `age = 25`:

```text
25 >= 18
   ↓
true
   ↓
"Adult"
```

---

# 5. Only one branch runs

This is worth making explicit.

Consider:

```kotlin
val age = 25

if (age >= 18) {
    println("Adult")
} else {
    println("Minor")
}
```

You will not get:

```text
Adult
Minor
```

You get only:

```text
Adult
```

The `if`/`else` structure chooses one path.

Think:

```text
if true
    → first branch

otherwise
    → second branch
```

Not:

```text
run both and see what happens
```

---

# 6. Conditions must produce a Boolean

This is directly connected to our previous lesson.

This is valid:

```kotlin
if (age >= 18) {
    println("Adult")
}
```

because:

```kotlin
age >= 18
```

produces a `Boolean`.

So does:

```kotlin
if (isLoggedIn) {
    println("Welcome")
}
```

assuming `isLoggedIn` is a `Boolean`.

But this:

```kotlin
val age = 20

if (age) {
    println("Adult")
}
```

doesn't work.

Why?

Because `age` is an `Int`, not a `Boolean`.

Kotlin does not treat:

```text
20
```

as automatically meaning:

```text
true
```

This is another example of Kotlin's strict type system.

---

# 7. You can put Boolean expressions directly into `if`

For example:

```kotlin
val age = 25
val hasId = true

if (age >= 18 && hasId) {
    println("Allowed")
}
```

The condition is:

```kotlin
age >= 18 && hasId
```

Let's evaluate it.

First:

```text
25 >= 18
   ↓
true
```

Then:

```text
true && true
   ↓
true
```

So the body executes.

You can make conditions as complicated as necessary, although there's a readability point we'll discuss later.

---

# 8. `else if`

What if there are more than two possibilities?

Suppose we want:

```text
90+ → A
80-89 → B
70-79 → C
60-69 → D
below 60 → F
```

We could write:

```kotlin
if (score >= 90) {
    println("A")
} else if (score >= 80) {
    println("B")
} else if (score >= 70) {
    println("C")
} else if (score >= 60) {
    println("D")
} else {
    println("F")
}
```

Now Kotlin evaluates the conditions **from top to bottom**.

Suppose:

```kotlin
val score = 85
```

First:

```text
85 >= 90
   ↓
false
```

Next:

```text
85 >= 80
   ↓
true
```

So `"B"` is printed.

Kotlin then stops evaluating this `if`/`else if` chain.

It doesn't continue and print `"C"` as well.

---

# 9. Order matters enormously

Consider:

```kotlin
val score = 85

if (score >= 60) {
    println("D or better")
} else if (score >= 80) {
    println("B or better")
}
```

What happens?

The first condition is:

```text
85 >= 60
   ↓
true
```

So the first branch executes.

Kotlin never reaches the `score >= 80` branch.

Therefore, in an `if`/`else if` chain:

> **The first true condition wins.**

This means the order of conditions can completely change the behavior of your program.

That's an important debugging skill.

---

# 10. Think of `else if` as a sequence of questions

This:

```kotlin
if (score >= 90) {
    ...
} else if (score >= 80) {
    ...
} else if (score >= 70) {
    ...
} else {
    ...
}
```

can be mentally read as:

```text
Is score >= 90?
    Yes → stop here.
    No  → ask next question.

Is score >= 80?
    Yes → stop here.
    No  → ask next question.

Is score >= 70?
    Yes → stop here.
    No  → use else.
```

That mental model will help you enormously when we begin writing more complicated business logic.

---

# 11. Nested `if`

An `if` can exist inside another `if`.

For example:

```kotlin
val age = 25
val hasId = true

if (age >= 18) {
    if (hasId) {
        println("Entry allowed")
    }
}
```

Here:

1. First check age.
    
2. If the person is old enough, check ID.
    
3. Only if both succeed, print the message.
    

Conceptually:

```text
age >= 18?
   │
   ├── false → stop
   │
   └── true
         ↓
      hasId?
       │
       ├── false → stop
       │
       └── true
              ↓
          Entry allowed
```

This works.

But often, we can express the same logic more cleanly:

```kotlin
if (age >= 18 && hasId) {
    println("Entry allowed")
}
```

This is one reason understanding Boolean operators is so useful.

---

# 12. `if` can have no `else`

There's no requirement to have an `else`.

For example:

```kotlin
if (temperature > 30) {
    println("It's hot")
}
```

If the condition is false, nothing happens.

That's perfectly valid.

So these are both valid forms:

```kotlin
if (condition) {
    ...
}
```

and:

```kotlin
if (condition) {
    ...
} else {
    ...
}
```

Use `else` when you actually need an alternative.

---

# 13. Here's something special about Kotlin: `if` is an expression

This is a very important Kotlin feature.

In some languages, `if` is primarily thought of as a statement.

In Kotlin, `if` can produce a value.

For example:

```kotlin
val age = 20

val message = if (age >= 18) {
    "Adult"
} else {
    "Minor"
}
```

Now:

```text
age >= 18
    ↓
true
    ↓
"Adult"
    ↓
message
```

So:

```kotlin
println(message)
```

prints:

```text
Adult
```

This is a major Kotlin idea.

We can essentially say:

> Evaluate this condition, and depending on the result, produce one value or another.

---

# 14. This is different from the way we first used `if`

Compare:

### `if` controlling execution

```kotlin
if (age >= 18) {
    println("Adult")
}
```

with:

### `if` producing a value

```kotlin
val message = if (age >= 18) {
    "Adult"
} else {
    "Minor"
}
```

The second one has an important structure:

```text
condition
   ↓
choose branch
   ↓
produce value
   ↓
store value
```

This expression-oriented style is used heavily in Kotlin.

---

# 15. The branches need to produce compatible values

Consider:

```kotlin
val age = 20

val message = if (age >= 18) {
    "Adult"
} else {
    "Minor"
}
```

Both branches produce `String`s.

So `message` is a `String`.

Now imagine:

```kotlin
val result = if (age >= 18) {
    "Adult"
} else {
    123
}
```

Now we're producing different types.

Kotlin's type system has to determine a type that can represent the result.

We'll explore exactly how Kotlin handles this when we study the type system more deeply.

The important thing for now is:

> When `if` is used as an expression, the branches are being used to produce a value.

---

# 16. What if there's no `else`?

This is an important distinction.

You can write:

```kotlin
if (age >= 18) {
    println("Adult")
}
```

But you generally can't use that as a value in the same way:

```kotlin
val message = if (age >= 18) {
    "Adult"
}
```

because what value would `message` get when the condition is false?

There is no alternative value.

This is one reason an `if` used to produce a value generally requires both branches.

Conceptually:

```text
if expression
    condition true  → value A
    condition false → value B
```

Without `else`, there is no `value B`.

---

# 17. Blocks have a result

Here's a slightly more advanced Kotlin idea.

Consider:

```kotlin
val result = if (true) {
    println("Hello")
    10
} else {
    20
}
```

What is `result`?

It is:

```text
10
```

Why?

Because the final expression of the chosen block becomes the value of that block.

So:

```kotlin
{
    println("Hello")
    10
}
```

does two things:

1. Prints `"Hello"`.
    
2. Produces the value `10`.
    

This idea will become increasingly important in Kotlin.

---

# 18. This makes Kotlin code very expressive

You can write:

```kotlin
val age = 25

val category = if (age < 13) {
    "Child"
} else if (age < 18) {
    "Teenager"
} else {
    "Adult"
}
```

Now `category` contains whichever string corresponds to the first matching condition.

For `age = 25`:

```text
25 < 13 → false
25 < 18 → false
else    → "Adult"
```

So:

```text
category → "Adult"
```

Then:

```kotlin
println(category)
```

prints:

```text
Adult
```

This is a very natural way to compute a value.

---

# 19. Let's look at a real example

Suppose we're calculating shipping cost.

```kotlin
val orderTotal = 2500

val shippingCost = if (orderTotal >= 2000) {
    0
} else {
    100
}

println(shippingCost)
```

The logic is:

```text
orderTotal >= 2000?
       │
       ├── true  → shippingCost = 0
       │
       └── false → shippingCost = 100
```

This is much more interesting than simply printing things because we're now **computing a program value based on a decision**.

That pattern appears everywhere in software.

---

# 20. Another example: maximum of two numbers

Suppose:

```kotlin
val a = 10
val b = 25
```

We want the larger value.

We could write:

```kotlin
val maximum = if (a > b) {
    a
} else {
    b
}
```

Since:

```text
10 > 25
   ↓
false
```

the result is `b`:

```text
maximum → 25
```

This is a nice example because it shows that `if` isn't inherently about printing messages.

It's about **choosing a result based on a condition**.

---

# 21. You can use variables inside conditions

For example:

```kotlin
val age = 21
val hasTicket = true
val isBanned = false

val canEnter =
    age >= 18 &&
    hasTicket &&
    !isBanned

if (canEnter) {
    println("Entry allowed")
} else {
    println("Entry denied")
}
```

This is often easier to understand than putting the entire expression directly into `if`.

There's no universal rule here.

Sometimes:

```kotlin
if (age >= 18 && hasTicket && !isBanned) {
```

is perfectly readable.

Sometimes giving the condition a meaningful name is better:

```kotlin
val canEnter = ...
```

Good programming involves making that judgment.

---

# 22. Don't over-nest your `if`s

You could write something absurd like:

```kotlin
if (condition1) {
    if (condition2) {
        if (condition3) {
            if (condition4) {
                println("Success")
            }
        }
    }
}
```

It's technically possible.

But now you have four levels of indentation just to understand one logical condition.

Often:

```kotlin
if (condition1 && condition2 && condition3 && condition4) {
    println("Success")
}
```

is easier.

There are cases where nested decisions genuinely make sense, so don't treat nesting as inherently wrong.

The principle is:

> **Structure the code so that the logic is easy to follow.**

---

# 23. A debugging technique: trace the branches

Consider:

```kotlin
val score = 72

if (score >= 90) {
    println("A")
} else if (score >= 80) {
    println("B")
} else if (score >= 70) {
    println("C")
} else {
    println("F")
}
```

Don't guess.

Trace it:

```text
score = 72

72 >= 90
   ↓
false

72 >= 80
   ↓
false

72 >= 70
   ↓
true

→ print C
```

This kind of reasoning will become extremely useful when debugging real applications.

---

# 24. A subtle logical issue

Consider:

```kotlin
val age = 25

if (age >= 18) {
    println("Adult")
} else if (age >= 21) {
    println("21+")
}
```

What prints?

You might expect:

```text
21+
```

But you'll get:

```text
Adult
```

because:

```text
25 >= 18
```

is already true.

The second branch is never reached.

If you need the 21+ case to be handled specifically, the order would need to change:

```kotlin
if (age >= 21) {
    println("21+")
} else if (age >= 18) {
    println("Adult")
}
```

Now the more specific condition gets checked first.

This is a great example of why understanding control flow is much more important than memorizing syntax.

---

# 25. Let's connect this back to expressions

Remember our previous lesson:

> An expression produces a value.

Now look at:

```kotlin
age >= 18
```

produces:

```text
true / false
```

Then `if` consumes that Boolean and chooses a path.

When used as an expression:

```kotlin
val category = if (age >= 18) "Adult" else "Minor"
```

the `if` itself produces a value.

So we're building a chain:

```text
age
 ↓
comparison
 ↓
Boolean
 ↓
if
 ↓
chosen value
 ↓
variable
```

For example:

```text
25
 ↓
25 >= 18
 ↓
true
 ↓
"Adult"
 ↓
category
```

This is a core Kotlin way of thinking.

---

# 26. Your exercises

Don't rush these. The point is to make yourself **trace the logic**.

### Exercise 1

Write a program that contains:

```kotlin
val age = ...
```

and prints:

```text
"Adult"
```

when the age is at least 18, otherwise:

```text
"Minor"
```

---

### Exercise 2

Write a program that checks whether a number is positive, negative, or zero.

For example:

```text
10  → Positive
-5  → Negative
0   → Zero
```

You should use `if` / `else if` / `else`.

---

### Exercise 3

Create a `score` variable and print:

```text
A → 90+
B → 80–89
C → 70–79
D → 60–69
F → below 60
```

Be careful about the order of your conditions.

---

### Exercise 4

Without running it, determine the output:

```kotlin
val x = 15

if (x > 10) {
    println("A")
} else if (x > 5) {
    println("B")
} else {
    println("C")
}
```

Then determine the output of:

```kotlin
val x = 15

if (x > 20) {
    println("A")
} else if (x > 10) {
    println("B")
} else if (x > 5) {
    println("C")
} else {
    println("D")
}
```

The second one is especially useful for understanding that **only the first matching branch executes**.

---

# 27. One challenge

Write this without searching for a solution:

You have:

```kotlin
val amount = 7500
val isMember = true
```

Your program should calculate a discount according to these rules:

```text
Member + amount >= 5000 → 20% discount
Member + amount < 5000  → 10% discount
Not a member            → 0% discount
```

Then calculate the final amount.

This combines nearly everything we've learned:

- variables
    
- `Int` / `Double`
    
- arithmetic
    
- comparison
    
- `&&`
    
- `if` / `else if` / `else`
    
- `if` as an expression
    

Try solving it yourself before looking at anything else.

---

# The most important idea from this lesson

Don't think of `if` merely as:

> "A way to conditionally run some code."

That's true, but incomplete.

A better understanding is:

> **`if` lets a program choose between different computational paths based on a Boolean expression.**

And in Kotlin, because `if` is an expression, it can also **produce a value**.

That gives us:

```kotlin
val result = if (condition) {
    valueA
} else {
    valueB
}
```

which is essentially:

```text
condition
   ↓
 ┌───────┐
 │ true? │
 └───┬───┘
     │
 ┌───┴────┐
 │        │
true     false
 │        │
 A        B
 └───┬────┘
     ↓
  result
```

We've now learned **straight-line execution + decisions**.

The next piece of control flow is **repetition**: how to tell the program to do something multiple times. We'll get into `while`, `for`, ranges, `downTo`, `until`, `step`, `break`, and `continue`, and along the way you'll start seeing why computers are so good at repetitive tasks.