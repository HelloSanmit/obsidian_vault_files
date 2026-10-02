
# Lesson 1 — What Are We Actually Doing When We Program?

Before we learn Kotlin syntax, I want you to understand what a program **is**.

This may sound basic, but it prevents a lot of confusion later. When people jump directly into syntax, they often learn:

> “Write this symbol here and that keyword there.”

They can produce code, but they don't really understand what the computer is doing.

We're going to build the mental model first.

---

## 1. What is programming?

At its simplest, programming is:

> **Giving a computer a precise sequence of instructions for transforming some starting state into a desired result.**

Suppose you want a program that calculates the area of a rectangle.

A human naturally thinks:

> “Area = length × width.”

A computer doesn't understand that sentence.

You have to express it in a language whose rules the computer's software tools understand:

```kotlin
val length = 10
val width = 5

val area = length * width
```

The important part isn't the syntax yet.

Conceptually, you've told the computer:

```text
1. Remember 10.
2. Remember 5.
3. Multiply them.
4. Remember the result as area.
```

Programming is largely the art of taking a problem that is vague to a human and making its solution **precise enough that a machine can execute it**.

---

# 2. Computers are not actually "smart" in the way programmers sometimes make them sound

Consider this:

```kotlin
val age = 20

if (age >= 18) {
    println("Adult")
}
```

A beginner might mentally read this as:

> “The computer knows that someone who is 18 or older is an adult.”

Not really.

The computer is following a set of mechanically defined operations.

Very roughly:

```text
Take the value stored in age
        ↓
Compare it with 18
        ↓
Get either true or false
        ↓
If the result is true
        ↓
execute println(...)
```

This distinction becomes extremely important when programs get complicated.

Programming isn't about convincing a computer to understand your intentions.

It's about **representing your intentions in a form that can be executed unambiguously**.

---

# 3. So where does Kotlin fit into this?

Kotlin is a **programming language**.

Just like humans use languages to communicate, programmers use programming languages to communicate instructions to computing systems.

Examples include:

```text
Kotlin
Java
C
C++
Python
JavaScript
Rust
Go
Swift
```

They have different syntax and different design philosophies, but they all solve a similar fundamental problem:

> How do humans express computational instructions in a precise form?

For us, that language is Kotlin.

---

# 4. Why can't we just write instructions in normal English?

Imagine writing:

```text
Take the user's age and if they're old enough, print that they're an adult.
```

The problem is that English contains enormous amounts of ambiguity.

What does "old enough" mean?

```text
18?
21?
16?
The legal age in which country?
```

A programming language deliberately eliminates much of that ambiguity.

For example:

```kotlin
if (age >= 18) {
    println("Adult")
}
```

Now the operation is precisely defined.

The computer doesn't have to interpret the meaning of "old enough."

It only has to perform a comparison.

---

# 5. But Kotlin code isn't what the CPU directly understands

This is an extremely important concept.

You write:

```kotlin
println("Hello")
```

Your CPU does **not** directly understand Kotlin.

The CPU ultimately executes machine instructions.

So there has to be a chain between your source code and the actual execution.

For our Kotlin journey, a useful simplified model is:

```text
Your Kotlin source code
        ↓
Kotlin compiler
        ↓
JVM-compatible code / bytecode
        ↓
JVM
        ↓
Machine-level execution
        ↓
CPU
```

Don't worry about the details yet.

The important thing is:

> **Kotlin is a human-friendly representation of computation, not the final form the processor executes.**

---

# 6. What's the JVM?

You've probably heard:

> Kotlin runs on the JVM.

The **JVM** stands for **Java Virtual Machine**.

Think of it, initially, as a machine that understands a particular intermediate form of code called **bytecode**.

So the Kotlin compiler can take something like:

```kotlin
val x = 10
val y = 20
println(x + y)
```

and translate it into JVM-compatible instructions.

The JVM then executes those instructions.

This is one of the reasons Kotlin can work so naturally with the huge Java ecosystem.

We'll eventually go much deeper into:

```text
Kotlin
   ↓
Compiler
   ↓
Bytecode
   ↓
JVM
   ↓
Operating system
   ↓
CPU
```

But **not today**.

For now, keep the picture in your head.

---

# 7. Kotlin is not the same thing as Android

This distinction will save you a lot of confusion.

You may have encountered Kotlin primarily because of Android development.

But:

```text
Kotlin ≠ Android
```

Kotlin is a programming language.

Android is an operating system/platform and development ecosystem.

You can use Kotlin for things such as:

```text
Android applications
Backend servers
Command-line programs
Desktop applications
JVM applications
Multiplatform applications
Libraries
Automation/tools
```

So we're going to learn **Kotlin itself first**.

Later, we can put Kotlin inside Android or another ecosystem.

---

# 8. Our first Kotlin program

Eventually you'll write something like this:

```kotlin
fun main() {
    println("Hello, Kotlin!")
}
```

This is tiny, but there are already several concepts hiding inside it.

Let's take it apart.

---

## `fun`

```kotlin
fun
```

This tells Kotlin that we're declaring a **function**.

A function is essentially a named piece of behavior.

Don't worry about the complete definition yet.

For now:

> `fun` introduces a function declaration.

---

## `main`

```kotlin
main
```

`main` is the entry point of this program.

It tells the environment:

> "Start executing the program here."

So:

```kotlin
fun main()
```

means, roughly:

> "Here is the main function from which execution begins."

---

## `{ ... }`

```kotlin
fun main() {
    ...
}
```

The braces define the function's body.

Everything inside them belongs to that function.

So:

```kotlin
fun main() {
    println("Hello")
    println("World")
}
```

means that the two `println` calls are part of `main`.

---

## `println`

```kotlin
println("Hello, Kotlin!")
```

`println` prints something and then moves to the next line.

We don't need to know yet how `println` itself is implemented.

Think of it as an already-provided function that we can call.

---

## `"Hello, Kotlin!"`

That's a **string**.

A string is text.

So:

```kotlin
println("Hello, Kotlin!")
```

means:

> Call `println` and give it the text `"Hello, Kotlin!"`.

---

# 9. There's a useful distinction here

Consider:

```kotlin
println("Hello")
```

There are several layers of understanding.

At the surface:

```text
"This prints Hello."
```

A little deeper:

```text
We are calling a function named println.
```

Deeper:

```text
We're passing a String argument to that function.
```

Deeper:

```text
The function eventually causes characters to be written to standard output.
```

And much deeper:

```text
The runtime and operating system eventually handle the underlying I/O.
```

A good programmer gradually becomes comfortable moving between these levels.

You **don't need to know everything about the bottom layer before using the top layer**.

That's one of the central skills we'll develop.

---

# 10. Code is built from smaller concepts

Look at this:

```kotlin
fun main() {
    val name = "Alice"
    println("Hello, $name")
}
```

Eventually you'll recognize:

```text
fun
    ↓
function declaration

main
    ↓
function name

val
    ↓
variable/reference declaration

name
    ↓
identifier

"Alice"
    ↓
String value

println(...)
    ↓
function call

"$name"
    ↓
string template
```

Right now, though, I don't expect you to know all of those.

That's deliberate.

We're going to build them one at a time.

---

# 11. One important idea: syntax vs meaning

Suppose I show you:

```kotlin
val age = 25
```

The **syntax** is the exact form:

```text
val + identifier + = + value
```

But the **meaning** is something more important:

> Create a reference named `age` associated with the value `25`.

We'll spend a lot of time learning to separate these two things.

Because when you're learning a programming language, there is always a temptation to memorize patterns:

```text
Oh, `val x = ...` means this.
```

Instead, I want you to eventually look at it and think:

> What entity exists here?  
> What type does it have?  
> What value does it refer to?  
> Who can access it?  
> Can it change?  
> What happens when this line executes?

That is much closer to actual programming.

---

# 12. Programs are transformations

Here's an idea I want you to keep.

Suppose:

```kotlin
val price = 100
val quantity = 3
val total = price * quantity
```

You can think of the program as transforming information:

```text
price = 100
quantity = 3

       ↓

price × quantity

       ↓

300

       ↓

total = 300
```

This way of thinking scales beautifully.

A much larger program may do:

```text
User input
    ↓
Validation
    ↓
Transformation
    ↓
Business logic
    ↓
Database
    ↓
Network
    ↓
Output
```

At every stage, information is being transformed or acted upon.

This is one of the most useful mental models you'll acquire.

---

# 13. Your first little experiment

When you eventually have Kotlin running, create:

```kotlin
fun main() {
    println("Hello, Kotlin!")
    println("I am learning programming.")
    println(10 + 20)
}
```

Notice something interesting.

The first two arguments are text:

```kotlin
"Hello, Kotlin!"
"I am learning programming."
```

But the third is an expression:

```kotlin
10 + 20
```

The computer evaluates that expression first and then gives the result to `println`.

So:

```kotlin
println(10 + 20)
```

produces:

```text
30
```

This is our first tiny glimpse of an important idea:

> **Expressions produce values.**

We're going to build almost the entire Kotlin language on top of this idea.

---

# 14. Try predicting the output before running code

This is a habit I want you to develop immediately.

Don't just execute code and look at what happens.

Predict first.

What does this print?

```kotlin
fun main() {
    println(10 + 5)
    println(10 - 5)
    println(10 * 5)
    println(10 / 5)
}
```

You should predict:

```text
15
5
50
2
```

Now change it:

```kotlin
fun main() {
    println(10 / 3)
}
```

Don't run it yet.

What do you expect?

Many beginners instinctively say:

```text
3.333333...
```

But that's **not** what we'll get here.

We'll use this to start understanding types and integer arithmetic in the next lessons.

The point isn't the answer itself.

The point is developing the habit:

> **Read → reason → predict → run → compare.**

That habit will become one of your strongest programming skills.

---

# 15. Programming is also problem decomposition

Suppose I ask:

> Write a program that calculates a student's average marks.

A beginner may see that as one giant task.

A programmer starts breaking it down:

```text
1. Obtain the marks.
2. Store them.
3. Add them.
4. Divide by the number of subjects.
5. Display the result.
```

Then perhaps:

```text
Get marks
    ↓
Store data
    ↓
Calculate sum
    ↓
Calculate average
    ↓
Print result
```

Then we turn those steps into code.

This process—**breaking a complicated problem into smaller pieces**—is one of the most important things I'll teach you.

Kotlin syntax is comparatively easy.

Problem decomposition is the deeper skill.

---

# 16. Our philosophy for this course

I want you to develop three abilities simultaneously.

### You should know Kotlin.

That means syntax, language features, standard library, coroutines, generics, Flow, and so on.

### You should understand programming.

That means state, data, control flow, abstraction, algorithms, types, functions, errors, concurrency, etc.

### You should learn to think like an engineer.

That means asking:

```text
Is this design understandable?
What happens if the input is invalid?
What happens under concurrency?
Can this fail?
Can this be tested?
Can another programmer maintain it?
```

Someone can memorize a lot of Kotlin and still be weak at the other two.

Our course will deliberately develop all three.

---

# Your first mental model

For now, remember this picture:

```text
             YOU
              │
              │ write
              ▼
        Kotlin source code
              │
              │ compiler translates
              ▼
        JVM-compatible code
              │
              │ executed by
              ▼
             JVM
              │
              ▼
      Operating system
              │
              ▼
             CPU
```

And conceptually, your program is doing:

```text
Input
  ↓
Process / transform
  ↓
Output
```

with **data, functions, decisions, repetition and state** appearing as the programs become more sophisticated.

---

# A small exercise — don't look up the answers yet

Try to answer these in your own words.

**1.** What is a programming language?

**2.** Why can't the CPU simply execute Kotlin source code directly?

**3.** What is the JVM?

**4.** In this program:

```kotlin
fun main() {
    println(10 + 20)
}
```

what is the function?

**5.** What is the difference between:

```kotlin
println("10 + 20")
```

and:

```kotlin
println(10 + 20)
```

**6.** Before running this:

```kotlin
fun main() {
    println(10 + 5 * 2)
}
```

predict the output.

Don't worry about getting everything right. In fact, **wrong predictions are useful**, because they expose where our mental model needs work.

---

## Your first note

For your Obsidian vault, I'd create:

```text
00 Foundations/
    01 — What Is Programming and Kotlin.md
```

You can keep this lesson there, but don't feel compelled to preserve every conversational sentence. As we go, the notes should become your **understanding of Kotlin**, not an archive of ChatGPT messages.

Next, we'll make Kotlin concrete: **install/run the toolchain, compile a real program, understand exactly what `fun main()` means, and then start variables and values.**