
# Lesson 15 — `when`: Kotlin's Powerful Decision Expression

We've already learned `if`, `else if`, and `else`.

For example:

```kotlin
val score = 82

if (score >= 90) {
    println("A")
} else if (score >= 80) {
    println("B")
} else {
    println("Below B")
}
```

This works perfectly well.

But as the number of possible cases grows, long `if`/`else if` chains can become awkward.

Kotlin's answer is:

```kotlin
when
```

And `when` is much more than a prettier `switch`.

It can compare values, test ranges, check types, combine conditions, and—very importantly—**produce a value**.

---

# 1. The basic `when`

Suppose:

```kotlin
val day = 3
```

We want to translate:

```text
1 → Monday
2 → Tuesday
3 → Wednesday
...
```

We can write:

```kotlin
when (day) {
    1 -> println("Monday")
    2 -> println("Tuesday")
    3 -> println("Wednesday")
    4 -> println("Thursday")
    5 -> println("Friday")
    6 -> println("Saturday")
    7 -> println("Sunday")
    else -> println("Invalid day")
}
```

If:

```text
day = 3
```

then the output is:

```text
Wednesday
```

The basic mental model is:

```text
take a value
    ↓
compare it against cases
    ↓
run the first matching branch
```

This is similar to an `if`/`else if` chain, but often cleaner when you're comparing one thing against several alternatives.

---

# 2. `when` can be an expression

This is where Kotlin becomes particularly interesting.

Instead of:

```kotlin
when (day) {
    1 -> println("Monday")
    2 -> println("Tuesday")
    ...
}
```

we can calculate a value:

```kotlin
val dayName = when (day) {
    1 -> "Monday"
    2 -> "Tuesday"
    3 -> "Wednesday"
    4 -> "Thursday"
    5 -> "Friday"
    6 -> "Saturday"
    7 -> "Sunday"
    else -> "Invalid"
}
```

Now:

```kotlin
println(dayName)
```

prints:

```text
Wednesday
```

So the `when` itself produces a value.

That's conceptually the same idea we saw with:

```kotlin
val category = if (age >= 18) {
    "Adult"
} else {
    "Minor"
}
```

Both `if` and `when` can be used to **compute a result**.

---

# 3. One value, many possible results

Think of:

```kotlin
val message = when (status) {
    1 -> "Starting"
    2 -> "Running"
    3 -> "Finished"
    else -> "Unknown"
}
```

as a little value-producing machine:

```text
             status
                ↓
        ┌───────────────┐
        │     when      │
        └───────────────┘
          ↓    ↓    ↓
       case 1 case 2 case 3
          ↓    ↓    ↓
       "Start" "Run" "Done"
```

Exactly one matching branch is selected.

---

# 4. Multiple values can match the same branch

Suppose we want:

```text
1, 2, 3 → Weekday
4, 5    → Something else
```

You don't necessarily need separate branches.

You can write:

```kotlin
when (day) {
    1, 2, 3 -> println("First three days")
    4, 5 -> println("Later days")
    else -> println("Other")
}
```

The comma means:

> Match any of these values.

So:

```text
1 → first branch
2 → first branch
3 → first branch
4 → second branch
5 → second branch
```

---

# 5. `when` with ranges

This is one of the places where `when` becomes very convenient.

Suppose we're grading a score:

```kotlin
val score = 84

val grade = when (score) {
    in 90..100 -> "A"
    in 80..89 -> "B"
    in 70..79 -> "C"
    in 60..69 -> "D"
    else -> "F"
}
```

The syntax:

```kotlin
in 90..100
```

means the value is inside that range.

For `84`:

```text
84 ∈ 90..100 → false
84 ∈ 80..89  → true
```

so:

```text
grade = "B"
```

This is often much easier to read than constructing a long sequence of comparisons.

---

# 6. `when` without a value

Here's something important.

You don't always need:

```kotlin
when (someValue)
```

You can write:

```kotlin
when {
    condition1 -> ...
    condition2 -> ...
    condition3 -> ...
    else -> ...
}
```

For example:

```kotlin
val age = 25

when {
    age < 13 -> println("Child")
    age < 18 -> println("Teenager")
    age < 60 -> println("Adult")
    else -> println("Senior")
}
```

This is conceptually similar to:

```kotlin
if (age < 13) {
    ...
} else if (age < 18) {
    ...
} else if (age < 60) {
    ...
} else {
    ...
}
```

So why use `when` here?

Because it can make a multi-case decision read more like a list of rules.

---

# 7. The order still matters

Remember our earlier lesson:

> The first matching branch wins.

That applies to `when` too.

Suppose:

```kotlin
val score = 95

when {
    score >= 60 -> println("Pass")
    score >= 90 -> println("A")
    else -> println("Fail")
}
```

You might expect `"A"`.

But you'll get:

```text
Pass
```

because:

```text
95 >= 60
↓
true
```

The first branch already matched.

The later branch is never considered.

If you want the specific case first:

```kotlin
when {
    score >= 90 -> println("A")
    score >= 60 -> println("Pass")
    else -> println("Fail")
}
```

Now `"A"` is selected.

This is the same logical issue we saw with `else if`.

---

# 8. `when` can check types

This connects directly to Lesson 14.

Suppose:

```kotlin
fun describe(value: Any): String {
    return when (value) {
        is String -> "String: $value"
        is Int -> "Integer: $value"
        is Boolean -> "Boolean: $value"
        else -> "Something else"
    }
}
```

Now:

```kotlin
describe("Kotlin")
```

matches:

```kotlin
is String
```

while:

```kotlin
describe(42)
```

matches:

```kotlin
is Int
```

This is powerful because Kotlin also smart-casts inside the corresponding branch.

So:

```kotlin
is String -> value.length
```

can use `value` as a `String`.

The broad:

```text
Any
```

has been narrowed to:

```text
String
```

for that branch.

---

# 9. `when` and multiple conditions

You can combine conditions inside a branch.

For example:

```kotlin
val age = 25
val hasId = true

when {
    age >= 18 && hasId -> println("Entry allowed")
    age >= 18 -> println("ID required")
    else -> println("Too young")
}
```

Now we're effectively expressing a set of rules:

```text
adult + ID
    → allowed

adult + no ID
    → ID required

not adult
    → too young
```

This can make business rules easier to read.

But, as always, if a condition becomes enormous, break it into meaningful variables or functions.

---

# 10. `when` can match exact values

Suppose:

```kotlin
val command = "start"

when (command) {
    "start" -> println("Starting...")
    "stop" -> println("Stopping...")
    "pause" -> println("Pausing...")
    else -> println("Unknown command")
}
```

This is a natural use of `when`.

We're comparing one value against several known alternatives.

---

# 11. A key difference from a chain of `if`s

Compare:

```kotlin
when (command) {
    "start" -> ...
    "stop" -> ...
    "pause" -> ...
}
```

with:

```kotlin
if (command == "start") {
    ...
} else if (command == "stop") {
    ...
} else if (command == "pause") {
    ...
}
```

They can represent the same logic.

But `when` emphasizes:

> **This one value can have one of these cases.**

That can communicate the structure of the problem better.

---

# 12. `when` doesn't have to compare against constants

You can use expressions.

For example:

```kotlin
when {
    number % 2 == 0 -> println("Even")
    else -> println("Odd")
}
```

Or:

```kotlin
when {
    text.isEmpty() -> println("Empty")
    text.length < 10 -> println("Short")
    else -> println("Long")
}
```

So there are really two broad styles:

### Value-oriented

```kotlin
when (x) {
    1 -> ...
    2 -> ...
    else -> ...
}
```

### Condition-oriented

```kotlin
when {
    condition1 -> ...
    condition2 -> ...
    else -> ...
}
```

Both are legitimate.

---

# 13. `when` branches can contain blocks

A branch isn't limited to one tiny expression.

For example:

```kotlin
when (command) {
    "start" -> {
        println("Preparing...")
        println("Starting...")
    }

    "stop" -> {
        println("Stopping...")
        println("Cleanup...")
    }

    else -> {
        println("Unknown command")
    }
}
```

This is useful when a case requires multiple operations.

And just like other blocks in Kotlin, the last expression can become the result when `when` is used as an expression.

---

# 14. Branches can produce values

For example:

```kotlin
val number = 5

val description = when {
    number < 0 -> "Negative"
    number == 0 -> "Zero"
    else -> "Positive"
}
```

The branches each produce a `String`.

So the `when` produces one `String`.

Conceptually:

```text
number
 ↓
when
 ├── negative → "Negative"
 ├── zero     → "Zero"
 └── positive → "Positive"
 ↓
description
```

Again, you're **computing a value through control flow**.

---

# 15. Exhaustiveness

This is one of the really important ideas in Kotlin.

Suppose you're dealing with a fixed set of possibilities.

Kotlin can sometimes determine that you've handled **all possible cases**.

That matters because if the cases are exhaustive, you don't need an arbitrary `else`.

We'll see the strongest version of this when we learn **enums and sealed classes**.

For example, imagine:

```kotlin
enum class Direction {
    NORTH,
    SOUTH,
    EAST,
    WEST
}
```

Then:

```kotlin
fun describe(direction: Direction): String {
    return when (direction) {
        Direction.NORTH -> "Up"
        Direction.SOUTH -> "Down"
        Direction.EAST -> "Right"
        Direction.WEST -> "Left"
    }
}
```

No `else` is needed because Kotlin knows those are the four possible enum values.

That is much more powerful than simply writing a switch-like statement.

---

# 16. Why is exhaustiveness useful?

Imagine someone later adds:

```kotlin
Direction.CENTER
```

to the enum.

Then the `when` above is no longer exhaustive.

Kotlin can alert you that your `when` needs to account for the new case.

This is a fantastic example of the type system helping prevent incomplete logic.

You can think of it as:

```text
possible states
      ↓
must handle each state
      ↓
compiler can help verify this
```

This becomes extremely powerful with sealed classes.

---

# 17. `when` + enums is a preview of state modeling

Suppose:

```kotlin
enum class TrafficLight {
    RED,
    YELLOW,
    GREEN
}
```

Then:

```kotlin
fun action(light: TrafficLight): String {
    return when (light) {
        TrafficLight.RED -> "Stop"
        TrafficLight.YELLOW -> "Prepare"
        TrafficLight.GREEN -> "Go"
    }
}
```

Now the code explicitly models:

```text
TrafficLight
    ├── RED
    ├── YELLOW
    └── GREEN
```

and:

```text
when
   ↓
one behavior for each state
```

This is a fundamental modeling technique.

Later we'll use it for things like:

```text
Loading
Success
Error
```

or:

```text
Authenticated
Unauthenticated
Suspended
```

This is where sealed types become especially useful.

---

# 18. `when` can combine type checking and conditions

For example:

```kotlin
fun describe(value: Any): String {
    return when {
        value is String && value.isEmpty() -> "Empty string"
        value is String -> "Non-empty string"
        value is Int && value < 0 -> "Negative integer"
        value is Int -> "Non-negative integer"
        else -> "Something else"
    }
}
```

Notice what is happening:

```text
value is String
```

narrows the type.

Then:

```text
value.isEmpty()
```

can safely be used in the same branch.

That's the type system interacting with control flow.

---

# 19. A subtle point: `when` can be more readable, but not always

Don't replace every `if` with `when`.

For example:

```kotlin
if (isLoggedIn) {
    showDashboard()
}
```

is perfectly natural.

Turning that into:

```kotlin
when {
    isLoggedIn -> showDashboard()
}
```

doesn't improve anything.

Likewise:

```kotlin
if (age >= 18) {
    ...
} else {
    ...
}
```

is often clearer than a `when`.

A useful intuition is:

> Use `when` when you're expressing **multiple cases or rules**.

Use `if` when you're expressing a **simple condition**.

Not a strict rule, but a good starting point.

---

# 20. `when` can use ranges with `in` and `!in`

For example:

```kotlin
val age = 25

when (age) {
    in 0..12 -> println("Child")
    in 13..17 -> println("Teenager")
    in 18..59 -> println("Adult")
    else -> println("Senior")
}
```

You can also use:

```kotlin
!in
```

to mean "not in".

For example:

```kotlin
when (score) {
    !in 0..100 -> println("Invalid score")
    in 90..100 -> println("A")
    in 80..89 -> println("B")
    else -> println("Other")
}
```

This is particularly useful for validation.

---

# 21. You can use multiple conditions in a `when` branch

A single branch can match several values:

```kotlin
when (month) {
    1, 2, 3 -> println("Q1")
    4, 5, 6 -> println("Q2")
    7, 8, 9 -> println("Q3")
    10, 11, 12 -> println("Q4")
    else -> println("Invalid month")
}
```

That's much easier to read than:

```kotlin
if (month == 1 || month == 2 || month == 3) {
    ...
}
```

when the cases naturally belong together.

---

# 22. `when` without an `else`

Whether you need `else` depends on whether Kotlin can prove the `when` is exhaustive.

For something like:

```kotlin
when (day) {
    1 -> ...
    2 -> ...
    3 -> ...
}
```

there are many possible `Int` values you haven't handled.

So if you're using this as an expression, you need a path for the other possibilities.

That's why:

```kotlin
else -> ...
```

is needed.

But with a closed set such as an enum:

```kotlin
enum class Direction {
    NORTH, SOUTH, EAST, WEST
}
```

you can potentially cover every case directly.

This is an important distinction:

```text
open-ended type
→ can't enumerate every possible value

closed set of known states
→ compiler can verify exhaustiveness
```

We'll build on this heavily later.

---

# 23. A practical example: command processing

Suppose:

```kotlin
val command = "delete"
```

Then:

```kotlin
when (command) {
    "create" -> createItem()
    "update" -> updateItem()
    "delete" -> deleteItem()
    "list" -> listItems()
    else -> println("Unknown command")
}
```

The code communicates a command vocabulary directly.

This is a very natural `when` use case.

You might encounter exactly this sort of pattern in:

- CLI applications
    
- event systems
    
- parsers
    
- UI actions
    
- state handling
    

---

# 24. A practical example: HTTP-style result handling

Imagine we have a status code:

```kotlin
val statusCode = 404
```

We could classify it:

```kotlin
val message = when (statusCode) {
    200 -> "Success"
    201 -> "Created"
    400 -> "Bad request"
    401 -> "Unauthorized"
    404 -> "Not found"
    in 500..599 -> "Server error"
    else -> "Other response"
}
```

Now we're mixing:

- exact matching
    
- ranges
    
- a fallback
    

That's a good demonstration of why `when` is more expressive than a simple switch-style construct.

---

# 25. One subtle idea: matching is not the same as equality

When you write:

```kotlin
when (x) {
    10 -> ...
}
```

you're asking whether `x` matches that case.

For ordinary values, this behaves conceptually like equality matching.

But `when` supports several different kinds of conditions:

```text
exact value
multiple values
range
type
arbitrary Boolean condition
```

So don't reduce `when` to:

> "It's just `==` written more neatly."

It's more general than that.

---

# 26. Let's trace one carefully

Consider:

```kotlin
val value: Any = 42

val result = when (value) {
    is String -> "String of length ${value.length}"
    is Int -> {
        if (value % 2 == 0) {
            "Even integer"
        } else {
            "Odd integer"
        }
    }
    is Boolean -> "Boolean"
    else -> "Other"
}
```

Let's trace it.

Initially:

```text
value : Any
value = 42
```

First case:

```text
is String
↓
false
```

Second:

```text
is Int
↓
true
```

Inside that branch, Kotlin knows:

```text
value : Int
```

So:

```text
42 % 2
↓
0

0 == 0
↓
true
```

Therefore the branch produces:

```text
"Even integer"
```

and that becomes:

```text
result = "Even integer"
```

This one example combines:

```text
Any
↓
type checking
↓
smart casting
↓
if
↓
arithmetic
↓
Boolean comparison
↓
when expression
```

That's exactly how these concepts begin to interact in real Kotlin.

---

# 27. Exercises

### Exercise 1

Given:

```kotlin
val number = 7
```

use `when` to print:

```text
Positive
Negative
Zero
```

depending on the number.

---

### Exercise 2

Given:

```kotlin
val score = 86
```

use `when` with ranges to print:

```text
A
B
C
D
F
```

according to the grading rules we've been using.

---

### Exercise 3

Create:

```kotlin
val day = 6
```

and use `when` to print the day name.

Handle invalid values.

---

### Exercise 4

Create:

```kotlin
val number = 14
```

Use `when` to print:

```text
Even
Odd
```

Don't use `if`.

---

### Exercise 5

Given:

```kotlin
val value: Any = ...
```

use `when` and `is` to print whether it is:

```text
String
Int
Boolean
Double
Something else
```

---

### Exercise 6

Create an enum:

```kotlin
enum class Direction {
    NORTH,
    SOUTH,
    EAST,
    WEST
}
```

Then use `when` to return an instruction such as:

```text
NORTH → "Move up"
SOUTH → "Move down"
EAST  → "Move right"
WEST  → "Move left"
```

Try doing this **without `else`**.

This will give you your first hands-on experience with exhaustive `when`.

---

# Challenge

Build a small command processor.

Start with:

```kotlin
val command = "..."
```

Support:

```text
add
remove
list
help
exit
```

Use `when`.

Then take it one step further:

```text
"add Alice"
"remove Alice"
"list"
"help"
"exit"
```

Now you have a tiny parsing problem.

You don't need to solve the full parser elegantly yet. The goal is to start seeing that `when` can become part of a larger decision system.

---

# The bigger picture

We've now expanded our control-flow toolkit:

```text
if
├── simple condition
├── else
├── else if
└── can produce a value

when
├── exact values
├── multiple values
├── ranges
├── arbitrary conditions
├── type checks
└── can produce a value
```

And something especially powerful is happening:

```text
Type information
      +
Control flow
      ↓
Smart narrowing
      ↓
More precise operations
```

For example:

```kotlin
when (value) {
    is String -> value.length
    is Int -> value * 2
}
```

The type system knows more inside each branch than it knew before the branch.

That's going to matter enormously when we get to **sealed classes and data modeling**.

---

The next lesson will be a little different. We've learned enough control flow and basic data manipulation to start looking at **`List`, `Set`, and `Map` properly and in much greater depth**—especially the important distinction between read-only and mutable collections, indexing, iteration, common collection APIs, and what Kotlin's collection abstractions actually mean.