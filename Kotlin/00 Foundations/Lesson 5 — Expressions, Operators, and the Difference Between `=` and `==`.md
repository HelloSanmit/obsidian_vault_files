
# Lesson 5 — Expressions, Operators, and the Difference Between `=` and `==`

So far we've learned that variables hold or refer to values, and that those values have types.

Now we're going to learn how we **do things with those values**.

You've already used:

```kotlin
10 + 20
```

but that's only the beginning.

We'll build up from arithmetic to comparisons and Boolean logic, because these are the ingredients we'll soon use to make programs **make decisions**.

---

## 1. What is an expression?

This is one of the most important concepts in Kotlin.

Consider:

```kotlin
10 + 20
```

This is an **expression**.

An expression is something that can be evaluated to produce a value.

For example:

```kotlin
10 + 20
```

produces:

```text
30
```

So:

```text
10 + 20
   ↓
  30
```

Another expression:

```kotlin
5 * 4
```

produces:

```text
20
```

And:

```kotlin
10 > 5
```

produces:

```text
true
```

That's important:

> An expression doesn't have to produce a number. It can produce any value.

For example:

```kotlin
"Alice"
```

is an expression producing a `String`.

```kotlin
true
```

produces a `Boolean`.

And:

```kotlin
10 > 5
```

produces a `Boolean`.

---

# 2. Why do I care about this distinction?

Because Kotlin is very **expression-oriented**.

Look at:

```kotlin
val x = 10 + 20
```

The right side:

```kotlin
10 + 20
```

is an expression.

It produces `30`.

Then:

```text
x ← 30
```

Likewise:

```kotlin
val result = 10 > 5
```

The expression:

```kotlin
10 > 5
```

produces:

```text
true
```

so:

```text
result → true
```

This idea will become extremely important when we get to `if`, `when`, lambdas, and functions.

---

# 3. Arithmetic operators

Let's start with the familiar ones.

### Addition

```kotlin
10 + 5
```

→ `15`

### Subtraction

```kotlin
10 - 5
```

→ `5`

### Multiplication

```kotlin
10 * 5
```

→ `50`

### Division

```kotlin
10 / 5
```

→ `2`

### Remainder

Kotlin uses `%` for the remainder:

```kotlin
10 % 3
```

→ `1`

Because:

```text
10 ÷ 3 = 3 remainder 1
```

So:

```kotlin
17 % 5
```

produces:

```text
2
```

And:

```kotlin
20 % 4
```

produces:

```text
0
```

The remainder operator will turn out to be extremely useful.

For example, checking whether a number is even:

```kotlin
number % 2 == 0
```

We'll unpack that expression later.

---

# 4. Operator precedence

Consider:

```kotlin
10 + 5 * 2
```

What should happen first?

If you evaluate strictly from left to right:

```text
10 + 5 = 15
15 × 2 = 30
```

But that's not how it works.

Multiplication has higher precedence than addition:

```text
5 × 2 = 10
10 + 10 = 20
```

So:

```kotlin
println(10 + 5 * 2)
```

prints:

```text
20
```

This is the same general mathematical convention you're familiar with.

---

## 5. Parentheses make your intention explicit

You can write:

```kotlin
10 + (5 * 2)
```

which gives `20`.

Or:

```kotlin
(10 + 5) * 2
```

which gives `30`.

Parentheses are useful not only for changing precedence, but for making the code easier for humans to read.

Compare:

```kotlin
val total = price + shipping * quantity
```

with:

```kotlin
val total = price + (shipping * quantity)
```

The second makes the intended grouping immediately visible.

---

# 6. Unary operators

You can also put an operator before a value.

For example:

```kotlin
val x = -10
```

The `-` here isn't subtracting something from something else.

It's making the value negative.

You can also use:

```kotlin
val x = 10
val y = -x
```

Similarly, `+x` exists, although it's much less interesting in everyday code.

---

# 7. Increment and decrement

You may encounter:

```kotlin
x++
x--
```

These represent incrementing and decrementing.

For example:

```kotlin
var x = 10

x++
println(x)
```

prints:

```text
11
```

And:

```kotlin
x--
```

decreases it by one.

Conceptually:

```kotlin
x++
```

is similar to:

```kotlin
x += 1
```

and:

```kotlin
x--
```

to:

```kotlin
x -= 1
```

There are subtleties around prefix/postfix behavior (`++x` vs `x++`), especially when they're used inside larger expressions. We'll come back to that rather than encouraging you to use them everywhere.

For now, the simple variable-update case is enough.

---

# 8. Now let's talk about comparison

Arithmetic gives us numbers.

But programs need to **make decisions**.

For that, we need comparisons.

Suppose:

```kotlin
val age = 20
```

We might want to ask:

> Is age greater than 18?

Kotlin allows:

```kotlin
age > 18
```

This is an expression.

And its result is:

```text
true
```

because `20 > 18`.

---

# 9. Comparison operators

You'll use these constantly:

```text
>    greater than
<    less than
>=   greater than or equal to
<=   less than or equal to
```

Examples:

```kotlin
10 > 5
```

→ `true`

```kotlin
10 < 5
```

→ `false`

```kotlin
10 >= 10
```

→ `true`

```kotlin
10 <= 9
```

→ `false`

All of these produce a `Boolean`.

So:

```text
comparison
   ↓
Boolean
```

For example:

```kotlin
val age = 20

val isAdult = age >= 18
```

Now:

```text
age → 20
age >= 18 → true
isAdult → true
```

That gives us a very powerful pattern:

> Use an expression to compute a Boolean value, then store that Boolean for later.

---

# 10. Now we reach `==`

Remember our earlier warning:

```text
=   ≠   ==
```

They're not the same thing.

### `=`

Assignment:

```kotlin
var age = 20
age = 21
```

You are giving the variable a value.

### `==`

Equality comparison:

```kotlin
age == 21
```

You're asking:

> Does `age` have the value 21?

That expression produces a `Boolean`.

For example:

```kotlin
val age = 21

println(age == 21)
println(age == 18)
```

Output:

```text
true
false
```

This distinction is absolutely fundamental.

---

# 11. Read code as questions

This is a useful mental habit.

Instead of seeing:

```kotlin
age == 18
```

as some mysterious syntax, mentally read:

> "Is age equal to 18?"

Similarly:

```kotlin
age >= 18
```

becomes:

> "Is age greater than or equal to 18?"

And:

```kotlin
score < 50
```

becomes:

> "Is score less than 50?"

This becomes especially useful once we start writing `if` statements.

---

# 12. What about `!=`?

`!=` means **not equal to**.

For example:

```kotlin
val age = 20

println(age != 18)
```

produces:

```text
true
```

because 20 isn't 18.

So now we have:

```text
==   equal
!=   not equal
```

---

# 13. Combining conditions

A single Boolean condition is useful.

But often you need several conditions together.

Imagine:

> The user can enter a restricted area only if they're at least 18 **and** they have a valid ticket.

We need:

```text
age >= 18
AND
hasTicket
```

Kotlin uses:

```kotlin
&&
```

for logical AND.

So:

```kotlin
age >= 18 && hasTicket
```

produces a Boolean.

---

# 14. `&&` — AND

Both sides must be `true`.

|A|B|A && B|
|---|---|---|
|true|true|true|
|true|false|false|
|false|true|false|
|false|false|false|

This makes sense:

> "You must satisfy condition A AND condition B."

For example:

```kotlin
val age = 25
val hasTicket = true

val allowed = age >= 18 && hasTicket
```

Both conditions are true, so:

```text
allowed → true
```

---

# 15. `||` — OR

Kotlin uses:

```kotlin
||
```

for logical OR.

At least one side must be true.

|A|B|A \| B|
|---|---|---|
|true|true|true|
|true|false|true|
|false|true|true|
|false|false|false|

For example:

```kotlin
val isAdmin = false
val isOwner = true

val canEdit = isAdmin || isOwner
```

This gives:

```text
true
```

because at least one condition is true.

---

# 16. `!` — NOT

The `!` operator reverses a Boolean.

```kotlin
!true
```

→ `false`

and:

```kotlin
!false
```

→ `true`

For example:

```kotlin
val isLoggedIn = true

println(!isLoggedIn)
```

prints:

```text
false
```

You can think of:

```kotlin
!condition
```

as:

> "The opposite of condition."

---

# 17. Let's combine everything

Suppose:

```kotlin
val age = 25
val hasTicket = true
```

Then:

```kotlin
age >= 18 && hasTicket
```

breaks down into:

```text
age >= 18
   ↓
true

hasTicket
   ↓
true

true && true
   ↓
true
```

So:

```kotlin
val allowed = age >= 18 && hasTicket
```

gives:

```text
allowed → true
```

That's a good example of expressions nested inside other expressions.

---

# 18. Expressions can be nested

Consider:

```kotlin
val result = (10 + 5) * 2
```

The inner expression:

```kotlin
10 + 5
```

produces:

```text
15
```

Then:

```text
15 * 2
```

produces:

```text
30
```

Likewise:

```kotlin
val allowed = age >= 18 && hasTicket
```

contains two expressions whose results are then combined.

You can think of a program as **building values out of other values**.

That idea is much more powerful than memorizing individual operators.

---

# 19. Short-circuit evaluation

Here's something slightly more advanced, but worth introducing now.

With:

```kotlin
A && B
```

Kotlin doesn't necessarily evaluate both sides.

If `A` is already `false`, then the entire expression must be false:

```text
false && ?
   ↓
false
```

So Kotlin can avoid evaluating `B`.

Likewise with:

```kotlin
A || B
```

if `A` is already `true`, the entire expression must be true:

```text
true || ?
  ↓
true
```

So Kotlin can skip evaluating `B`.

This is called **short-circuit evaluation**.

You don't need to memorize the implementation details yet, but you'll eventually need to understand why this matters when expressions have side effects.

---

# 20. `&&` and `||` are not just punctuation

Suppose:

```kotlin
val age = 15
val hasParent = true
```

Then:

```kotlin
age >= 18 && hasParent
```

is:

```text
false && true
```

which is:

```text
false
```

But:

```kotlin
age >= 18 || hasParent
```

becomes:

```text
false || true
```

which is:

```text
true
```

That difference will control the behavior of our programs.

---

# 21. Operator precedence with Boolean logic

Consider:

```kotlin
a || b && c
```

`&&` has higher precedence than `||`.

So it means:

```kotlin
a || (b && c)
```

not:

```kotlin
(a || b) && c
```

You _can_ learn all of Kotlin's operator precedence rules, but in real code I'd rather see explicit parentheses when there's any realistic chance of confusion.

For example:

```kotlin
isAdmin || (isOwner && isActive)
```

is much easier to reason about than leaving the grouping entirely to precedence rules.

---

# 22. Comparing strings

We don't only compare numbers.

We can compare strings too:

```kotlin
val name = "Alice"

println(name == "Alice")
```

→ `true`

and:

```kotlin
println(name == "Bob")
```

→ `false`

Here `==` is checking **structural equality**.

That phrase will matter later when we study objects.

For now, the practical idea is:

```kotlin
name == "Alice"
```

asks whether the two strings have equal content.

---

# 23. A common beginner mistake

Don't write:

```kotlin
if (age = 18) {
```

when we get to conditions.

That's trying to use assignment where a Boolean comparison is needed.

The comparison would be:

```kotlin
age == 18
```

So:

```text
=    → put a value somewhere
==   → compare values
```

Burn that distinction into your brain now. You'll see this mistake constantly in beginner code.

---

# 24. Arithmetic + comparison + Boolean logic

Now let's write something that looks much more like a real program.

```kotlin
fun main() {
    val age = 22
    val hasId = true

    val isOldEnough = age >= 18
    val canEnter = isOldEnough && hasId

    println(isOldEnough)
    println(canEnter)
}
```

Trace it:

```text
age → 22
hasId → true

age >= 18
   ↓
true

true && true
   ↓
true
```

So:

```text
true
true
```

Notice how we're gradually moving from raw numbers toward **logic**.

That's exactly what we need before learning control flow.

---

# 25. There's another operator you'll encounter: `in`

Kotlin has some operators specifically designed around its language features.

One is:

```kotlin
in
```

For example:

```kotlin
val x = 5

println(x in 1..10)
```

This asks whether `x` is inside the range from 1 through 10.

It produces:

```text
true
```

And:

```kotlin
println(x in 10..20)
```

produces:

```text
false
```

We're not going to deeply study ranges today. I just want you to see that operators in Kotlin aren't limited to arithmetic.

We'll use `in` much more when we reach ranges and `when`.

---

# 26. An important way to read code

Look at:

```kotlin
val canEnter = age >= 18 && hasTicket
```

Instead of seeing it as:

> `val`, some symbols, some variables...

try to read the **meaning**:

> Create `canEnter`, whose value is determined by whether the age is at least 18 and the person has a ticket.

That's the direction we're aiming for.

Eventually, you shouldn't mentally translate every Kotlin character into English.

You should understand the code at the level of its **logic**.

---

# 27. Let's trace a slightly harder one

```kotlin
fun main() {
    val age = 17
    val hasPermission = true
    val isBanned = false

    val canEnter =
        age >= 18 && hasPermission && !isBanned

    println(canEnter)
}
```

Let's calculate.

First:

```text
age >= 18
   ↓
false
```

Then:

```text
hasPermission
   ↓
true
```

Then:

```text
!isBanned
```

Since:

```text
isBanned = false
```

we get:

```text
!false
 ↓
true
```

So the whole expression is:

```text
false && true && true
```

which becomes:

```text
false
```

Therefore:

```text
canEnter → false
```

Even though the person has permission and isn't banned, they don't meet the age requirement.

---

# 28. A very useful debugging technique

Whenever a Boolean expression gets complicated, **break it into intermediate values**.

Instead of:

```kotlin
val canEnter =
    age >= 18 && hasPermission && !isBanned
```

temporarily write:

```kotlin
val isOldEnough = age >= 18
val hasValidPermission = hasPermission
val isNotBanned = !isBanned

val canEnter =
    isOldEnough &&
    hasValidPermission &&
    isNotBanned
```

This may look verbose, but it makes debugging much easier.

Once you understand the logic, you can decide whether the original compact form is clearer.

Good programming isn't about minimizing the number of lines.

It's about making the logic easy to understand.

---

# 29. Your mental model is expanding

We started with:

```text
values
```

Then:

```text
values
  ↓
expressions
  ↓
new values
```

Now:

```text
numbers
  ↓
arithmetic expressions
  ↓
numbers

values
  ↓
comparison expressions
  ↓
Boolean

Booleans
  ↓
logical expressions
  ↓
Boolean
```

For example:

```text
10 + 20
   ↓
30

30 > 18
   ↓
true

true && false
   ↓
false
```

This is the foundation upon which `if` and `when` are built.

---

# 30. Your exercises

Don't just read these. **Predict first, then run them.**

### Exercise 1

What does this print?

```kotlin
fun main() {
    println(10 + 5 * 2)
    println((10 + 5) * 2)
    println(20 % 3)
}
```

### Exercise 2

What is the value of `result`?

```kotlin
val a = 10
val b = 20

val result = a < b
```

### Exercise 3

What does this print?

```kotlin
val age = 20

println(age == 20)
println(age != 20)
println(age >= 18)
println(age < 18)
```

### Exercise 4

Work this one out manually:

```kotlin
val a = true
val b = false

println(a && b)
println(a || b)
println(!a)
println(!b)
```

### Exercise 5 — important

What is wrong with this?

```kotlin
var score = 100

score == 200

println(score)
```

Does `score` become `200`?

And why?

---

# One challenge for you

Try to write this program yourself:

> Create a variable containing someone's age. Create another variable indicating whether they have a valid ID. Then create a Boolean called `canEnter` which is true only when the person is at least 18 **and** has a valid ID. Print `canEnter`.

Don't copy a solution. Try constructing it from the pieces you've learned.

---

## One thing to take away from this lesson

The deepest concept here isn't actually `+`, `-`, `&&`, or `==`.

It's this:

> **An expression is evaluated to produce a value.**

For example:

```kotlin
10 + 5
```

produces a number.

```kotlin
age >= 18
```

produces a Boolean.

```kotlin
age >= 18 && hasId
```

produces another Boolean.

And soon:

```kotlin
if (age >= 18) {
    ...
}
```

will use that Boolean to decide **which code gets executed**.

That's our next major step: **control flow and `if`**.