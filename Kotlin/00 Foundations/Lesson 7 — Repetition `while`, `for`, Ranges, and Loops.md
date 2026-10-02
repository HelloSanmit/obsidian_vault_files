
# Lesson 7 — Repetition: `while`, `for`, Ranges, and Loops

So far, our programs can:

- store information,
    
- calculate values,
    
- make decisions.
    

But there's a huge capability still missing.

Suppose I ask a computer to print this:

```text
1
2
3
4
5
6
7
8
9
10
```

We _could_ write:

```kotlin
println(1)
println(2)
println(3)
...
```

But imagine printing one million numbers.

This is where **loops** come in.

A loop tells the computer:

> **Repeat some piece of work.**

And this is one of the fundamental ideas in programming.

---

# 1. The basic idea behind a loop

Think about a task like:

> Keep asking the user for a password until they enter the correct one.

That's inherently repetitive:

```text
Ask for password
    ↓
Correct?
 ┌──┴───┐
Yes    No
 ↓      ↓
Stop   Ask again
         ↓
       repeat
```

Or:

> Process every item in a list.

```text
First item
   ↓
Process
   ↓
Next item
   ↓
Process
   ↓
Next item
   ↓
...
```

A loop gives us that repetition.

---

# 2. Your first loop: `while`

The simplest Kotlin loop is:

```kotlin
while (condition) {
    // repeated code
}
```

For example:

```kotlin
var count = 1

while (count <= 5) {
    println(count)
    count++
}
```

Output:

```text
1
2
3
4
5
```

Let's understand this properly rather than memorizing the syntax.

---

# 3. What actually happens?

Start with:

```text
count = 1
```

Kotlin evaluates:

```text
count <= 5
```

That's:

```text
1 <= 5
↓
true
```

So the body executes:

```text
print 1
```

Then:

```text
count++
```

so now:

```text
count = 2
```

Kotlin goes back to the condition:

```text
2 <= 5
↓
true
```

Print 2.

Then count becomes 3.

And so on.

Eventually:

```text
count = 6
```

Now:

```text
6 <= 5
↓
false
```

The loop stops.

So the mental model is:

```text
       ┌───────────────┐
       │ Check condition│
       └───────┬───────┘
               ↓
           true?
          /     \
        yes      no
         ↓        ↓
      run body   stop
         ↓
      repeat
         │
         └──────→ condition
```

That's a loop.

---

# 4. The condition controls the loop

Consider:

```kotlin
var count = 1

while (count <= 5) {
    println(count)
    count++
}
```

The condition is:

```kotlin
count <= 5
```

The body is:

```kotlin
{
    println(count)
    count++
}
```

The condition answers:

> "Should I execute another iteration?"

That's the key idea.

An **iteration** simply means one execution of the loop body.

In our example, there are five iterations.

---

# 5. What happens if the condition is false from the beginning?

For example:

```kotlin
var count = 10

while (count <= 5) {
    println(count)
    count++
}
```

The condition:

```text
10 <= 5
↓
false
```

So the body doesn't execute even once.

Therefore:

```text
(no output)
```

This gives us an important property of `while`:

> A `while` loop can execute **zero times**.

The condition is checked before each iteration.

---

# 6. Be careful: your loop needs a way to eventually stop

Look at this:

```kotlin
var count = 1

while (count <= 5) {
    println(count)
}
```

What's wrong?

`count` never changes.

So:

```text
count = 1
1 <= 5 → true
1 <= 5 → true
1 <= 5 → true
...
```

Forever.

You've created an **infinite loop**.

The program will keep running until something externally stops it.

This is one of the most common beginner mistakes.

Whenever you write a `while` loop, ask:

> **What causes the condition to eventually become false?**

In:

```kotlin
while (count <= 5) {
    println(count)
    count++
}
```

the answer is:

```text
count++
```

because eventually `count` reaches 6.

---

# 7. A loop is really state changing over time

This is a useful way to think about loops.

Initially:

```text
count = 1
```

After one iteration:

```text
count = 2
```

Then:

```text
count = 3
```

Then:

```text
count = 4
```

Then:

```text
count = 5
```

Then:

```text
count = 6
```

Now the condition fails.

So a loop often looks conceptually like:

```text
State
 ↓
Check
 ↓
Transform state
 ↓
Check
 ↓
Transform state
 ↓
...
```

This is a very powerful mental model because later we'll see loops, recursion, state machines, coroutines, and reactive systems all involve variations of **state changing over time**.

---

# 8. `while` is particularly useful when you don't know how many times you'll repeat

Consider a login system.

You don't know whether the user will enter the correct password on attempt 1, 2, or 5.

So:

```kotlin
while (!isLoggedIn) {
    // try again
}
```

makes conceptual sense.

Or:

> Keep reading numbers until the user enters 0.

```kotlin
var number = -1

while (number != 0) {
    // read another number
}
```

You don't know the iteration count beforehand.

That's where `while` naturally fits.

---

# 9. What about a loop where we _do_ know the values?

Suppose we want:

```text
1
2
3
4
5
```

We could use `while`, but Kotlin gives us a much nicer tool:

```kotlin
for
```

---

# 10. Your first `for` loop

```kotlin
for (i in 1..5) {
    println(i)
}
```

Output:

```text
1
2
3
4
5
```

That's already much cleaner.

Read it as:

> For each `i` in the range from 1 to 5, execute this body.

There's a lot hidden inside that line, so let's unpack it.

---

# 11. What is `1..5`?

This creates a **range**.

```kotlin
1..5
```

means:

```text
1, 2, 3, 4, 5
```

So:

```kotlin
for (i in 1..5)
```

means:

```text
i = 1
i = 2
i = 3
i = 4
i = 5
```

and the body runs once for each value.

So this:

```kotlin
for (i in 1..5) {
    println(i)
}
```

is essentially expressing:

> Iterate over these values, one at a time.

---

# 12. The variable `i`

Here:

```kotlin
for (i in 1..5)
```

`i` is the variable representing the **current value** of the iteration.

During the first iteration:

```text
i = 1
```

During the second:

```text
i = 2
```

and so on.

So:

```kotlin
for (i in 1..5) {
    println(i)
}
```

produces:

```text
1
2
3
4
5
```

You don't manually increment `i`.

The `for` loop handles iteration over the range for you.

---

# 13. `for` isn't really about numbers

This is important.

Beginners sometimes learn:

```kotlin
for (i in 1..10)
```

and conclude:

> "A `for` loop is for counting."

Not really.

A better understanding is:

> **A `for` loop iterates over something.**

A range is one thing you can iterate over.

Later, we'll iterate over:

- lists
    
- sets
    
- strings
    
- maps
    
- sequences
    
- custom objects
    

For example:

```kotlin
for (name in names) {
    println(name)
}
```

This isn't counting at all.

It's visiting each element.

That idea becomes extremely important when we study collections.

---

# 14. `until`

Suppose we want:

```text
0
1
2
3
4
```

We could write:

```kotlin
for (i in 0..4) {
    println(i)
}
```

But Kotlin also gives us:

```kotlin
for (i in 0 until 5) {
    println(i)
}
```

`until` excludes the ending value.

So:

```text
0 until 5
```

means:

```text
0, 1, 2, 3, 4
```

whereas:

```text
0..5
```

means:

```text
0, 1, 2, 3, 4, 5
```

This distinction becomes very useful when working with indexes.

---

# 15. `downTo`

You can also count backward.

```kotlin
for (i in 5 downTo 1) {
    println(i)
}
```

Output:

```text
5
4
3
2
1
```

That's much clearer than manually managing a `var` and decrementing it.

---

# 16. `step`

What if you want:

```text
2
4
6
8
10
```

You can write:

```kotlin
for (i in 2..10 step 2) {
    println(i)
}
```

The values are:

```text
2
4
6
8
10
```

Or:

```kotlin
for (i in 10 downTo 2 step 2) {
    println(i)
}
```

gives:

```text
10
8
6
4
2
```

So Kotlin's range/progression syntax lets you express iteration patterns quite naturally.

---

# 17. A useful comparison

These two programs can accomplish similar tasks.

### Using `while`

```kotlin
var i = 1

while (i <= 5) {
    println(i)
    i++
}
```

### Using `for`

```kotlin
for (i in 1..5) {
    println(i)
}
```

The second is shorter because the programmer is saying:

> "I want to iterate over these values."

rather than manually managing the loop state.

That's a general principle:

> **Use the abstraction that expresses your intention most directly.**

---

# 18. Let's combine loops with `if`

Now programs become considerably more interesting.

Suppose we want to print only even numbers from 1 to 10:

```kotlin
for (i in 1..10) {
    if (i % 2 == 0) {
        println(i)
    }
}
```

Output:

```text
2
4
6
8
10
```

Let's trace one iteration.

Suppose:

```text
i = 3
```

Then:

```text
3 % 2
↓
1
```

So:

```text
1 == 0
↓
false
```

Nothing is printed.

For:

```text
i = 4
```

we get:

```text
4 % 2
↓
0

0 == 0
↓
true
```

So `4` gets printed.

We've just combined:

```text
iteration
+
condition
```

This pattern is everywhere in programming.

---

# 19. Looping while doing calculations

Suppose we want the sum of numbers from 1 to 5.

We can do:

```kotlin
var sum = 0

for (i in 1..5) {
    sum += i
}

println(sum)
```

Let's trace:

```text
sum = 0

i = 1 → sum = 1
i = 2 → sum = 3
i = 3 → sum = 6
i = 4 → sum = 10
i = 5 → sum = 15
```

Final:

```text
15
```

This introduces an important pattern called an **accumulator**.

We start with some value:

```text
sum = 0
```

Then repeatedly update it:

```text
sum = sum + currentValue
```

Eventually we get the accumulated result.

We'll encounter this concept again when we study collection functions like `fold`.

---

# 20. Another example: finding a value

Suppose:

```kotlin
val target = 7

for (i in 1..10) {
    if (i == target) {
        println("Found it!")
    }
}
```

When `i` reaches 7:

```text
7 == 7
↓
true
```

and we print:

```text
Found it!
```

But notice something:

The loop continues after finding it.

Sometimes that's exactly what you want.

Sometimes it isn't.

That's where `break` comes in.

---

# 21. `break`

`break` immediately exits the loop.

For example:

```kotlin
for (i in 1..10) {
    if (i == 7) {
        break
    }

    println(i)
}
```

Output:

```text
1
2
3
4
5
6
```

When `i` becomes 7:

```text
i == 7
↓
true
↓
break
↓
leave loop
```

So `7` is never printed.

The program jumps out of the loop entirely.

---

# 22. `break` with a search

This is a common pattern:

```kotlin
for (i in 1..100) {
    if (i == 42) {
        println("Found")
        break
    }
}
```

Once the target is found, there's no reason to continue searching.

Conceptually:

```text
search
 ↓
found?
 ↓
yes → stop searching
```

That's a legitimate use for `break`.

---

# 23. `continue`

`continue` means something different.

It says:

> Skip the rest of this iteration and move to the next one.

For example:

```kotlin
for (i in 1..10) {
    if (i % 2 != 0) {
        continue
    }

    println(i)
}
```

Output:

```text
2
4
6
8
10
```

Let's examine `i = 3`.

```text
3 % 2 != 0
↓
true
↓
continue
```

So Kotlin skips:

```kotlin
println(i)
```

and moves to the next iteration.

But unlike `break`, the loop itself continues.

So:

```text
break
↓
leave loop completely
```

whereas:

```text
continue
↓
skip current iteration
↓
continue looping
```

---

# 24. Don't use `continue` when an ordinary `if` is clearer

You could write:

```kotlin
for (i in 1..10) {
    if (i % 2 == 0) {
        println(i)
    }
}
```

instead of:

```kotlin
for (i in 1..10) {
    if (i % 2 != 0) {
        continue
    }

    println(i)
}
```

Both can work.

But the first may be easier to read because it directly says:

> If the number is even, print it.

Again, this isn't about minimizing lines.

It's about expressing the logic clearly.

---

# 25. `while` vs `do while`

There's one more loop worth understanding.

Normally:

```kotlin
while (condition) {
    ...
}
```

checks the condition **before** executing the body.

But:

```kotlin
do {
    ...
} while (condition)
```

executes the body first, then checks the condition.

For example:

```kotlin
var x = 10

do {
    println(x)
} while (x < 5)
```

What happens?

It prints:

```text
10
```

Even though:

```text
10 < 5
↓
false
```

Why?

Because the body runs once **before** the condition is checked.

So:

```text
while
→ possibly zero executions

do...while
→ at least one execution
```

---

# 26. A real use for `do while`

Suppose we need to ask the user for a valid number.

The user must enter something at least once:

```text
Ask user
  ↓
Is it valid?
  ↓
No → ask again
Yes → continue
```

That's naturally represented by `do while`.

We'll build interactive programs like this later.

---

# 27. Nested loops

You can put one loop inside another.

For example:

```kotlin
for (i in 1..3) {
    for (j in 1..3) {
        println("$i, $j")
    }
}
```

Output:

```text
1, 1
1, 2
1, 3
2, 1
2, 2
2, 3
3, 1
3, 2
3, 3
```

Think of it like an outer loop and an inner loop:

```text
i = 1
    j = 1
    j = 2
    j = 3

i = 2
    j = 1
    j = 2
    j = 3

i = 3
    j = 1
    j = 2
    j = 3
```

The inner loop runs completely for every iteration of the outer loop.

Nested loops are useful for things like:

- grids
    
- tables
    
- matrices
    
- combinations
    
- some searching algorithms
    

But they can also become computationally expensive, which we'll study when we get into algorithms and complexity.

---

# 28. An important beginner concept: don't confuse the variable with the loop

Look at:

```kotlin
for (i in 1..5) {
    println(i)
}
```

Some beginners think:

> "`i` is somehow a permanent variable that gets updated."

It's better to think:

> The `for` loop supplies each successive value from the thing being iterated over.

The important abstraction isn't really:

```text
i++
```

It's:

```text
give me the next element
```

That's why the same `for` syntax can work with things that aren't numbers.

For example:

```kotlin
val names = listOf("Alice", "Bob", "Charlie")

for (name in names) {
    println(name)
}
```

Conceptually:

```text
name = "Alice"
name = "Bob"
name = "Charlie"
```

We'll eventually learn exactly why this works by studying **iterators** and collections.

---

# 29. A first look at strings as iterable data

Even a string can be iterated character by character:

```kotlin
for (character in "Kotlin") {
    println(character)
}
```

Conceptually:

```text
K
o
t
l
i
n
```

This is a beautiful example of why it is better to understand:

> `for` iterates over elements

rather than:

> `for` counts numbers.

We'll come back to iteration when we study collections.

---

# 30. Looping isn't just "doing the same thing"

Here's a more useful mental model.

A loop typically has:

```text
1. Some collection/range/state
2. A condition or stopping point
3. Work performed during each iteration
4. Some transition toward the next iteration
```

For example:

```kotlin
var count = 1

while (count <= 5) {
    println(count)
    count++
}
```

has:

```text
state:
count

condition:
count <= 5

work:
println(count)

transition:
count++
```

That model is more useful than just memorizing `while` syntax.

---

# 31. Let's write a slightly more realistic program

Imagine a simple score counter.

```kotlin
var score = 0

for (round in 1..5) {
    score += round * 10
}

println(score)
```

Let's trace:

```text
round = 1 → score = 10
round = 2 → score = 30
round = 3 → score = 60
round = 4 → score = 100
round = 5 → score = 150
```

So:

```text
score = 150
```

Notice how naturally loops let us express repeated computation.

---

# 32. Another useful example: multiplication table

```kotlin
val number = 7

for (i in 1..10) {
    println("$number × $i = ${number * i}")
}
```

Output:

```text
7 × 1 = 7
7 × 2 = 14
7 × 3 = 21
...
7 × 10 = 70
```

We've combined several things now:

- variables
    
- `for`
    
- ranges
    
- arithmetic
    
- string templates
    

This is beginning to look much more like a real program.

---

# 33. Your exercises

### Exercise 1 — counting

Print the numbers from 1 to 20 using a `for` loop.

Then print them backwards from 20 to 1.

---

### Exercise 2 — even numbers

Print every even number between 1 and 50.

Don't manually write the numbers.

---

### Exercise 3 — sum

Calculate the sum of:

```text
1 + 2 + 3 + ... + 100
```

using a loop.

Before running the program, calculate what you expect the answer to be.

---

### Exercise 4 — multiples

Print all multiples of 5 between 1 and 100.

Try solving this first with an `if` and `%`.

Then see whether you can express it directly using `step`.

---

### Exercise 5 — `while`

Create:

```kotlin
var number = 10
```

and use a `while` loop to count down:

```text
10
9
8
...
1
```

---

### Exercise 6 — `break`

Search through numbers 1 to 100.

When you reach 37, print:

```text
Found 37
```

and stop the loop.

---

### Exercise 7 — `continue`

Print numbers from 1 to 20, but skip all multiples of 3.

---

# 34. A challenge

Write a program that prints all numbers from 1 to 100, but:

- prints `"Fizz"` for multiples of 3,
    
- prints `"Buzz"` for multiples of 5,
    
- prints `"FizzBuzz"` for multiples of both,
    
- otherwise prints the number.
    

For example:

```text
1
2
Fizz
4
Buzz
Fizz
7
8
Fizz
Buzz
...
```

There's an important subtlety here:

What condition should you check **first**?

Think about the number 15.

It's divisible by 3 **and** 5.

So the order of your conditions matters.

This combines almost everything we've learned so far.

---

# One deeper idea to take away

Loops aren't really about syntax like:

```kotlin
while
for
break
continue
```

Those are just the tools.

The underlying idea is:

> **A program can repeatedly perform computation while moving through a sequence of values or changing state toward some stopping condition.**

So we can think of our program's capabilities growing:

```text
Values
   ↓
Expressions
   ↓
Decisions
   ↓
Repetition
```

And those four ideas already let you build surprisingly substantial programs.

But we've been writing everything inside `main()`.

Imagine this:

```kotlin
fun main() {
    // 300 lines of code
}
```

That would become a nightmare to maintain.

We need a way to **break a program into named, reusable pieces of behavior**.

That's exactly what functions are for.

We've technically seen `main()` since Lesson 1, but we've only treated it as the place where the program begins.

The next lesson will finally open up **functions properly**: parameters, return values, local variables, why functions are one of the most important ideas in programming, and how to turn a large problem into small understandable pieces.