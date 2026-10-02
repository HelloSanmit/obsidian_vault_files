
# Kotlin Lesson 2 — From Kotlin Code to a Running Program

Last lesson, we built the conceptual picture:

```text
You
 ↓
Kotlin source code
 ↓
Compiler
 ↓
JVM-compatible code
 ↓
JVM
 ↓
Computer
```

Today we're going to actually walk through that process.

The objective is not simply to make "Hello World" appear on your screen. It's to understand what a **project**, **source file**, **function**, **compiler**, **JDK**, and **run configuration** are doing around that tiny program.

---

## 1. Before Kotlin: what do we need?

For the Kotlin/JVM programs we're going to write, there are three important pieces:

```text
IDE
JDK
Kotlin
```

### IDE

An **IDE** (Integrated Development Environment) is the application in which we write and manage code.

We'll use **IntelliJ IDEA** for this course.

IntelliJ IDEA has first-class Kotlin support, and the Kotlin plugin is bundled with the IDE. The current JetBrains documentation also provides a dedicated Kotlin project wizard.

Think of the IDE as our workshop.

It gives us:

- the editor
    
- syntax highlighting
    
- error detection
    
- autocomplete
    
- debugging
    
- project/file management
    
- build and run controls
    

Without an IDE, we could write Kotlin in a plain text editor and use the compiler from a terminal. We'll eventually do that too, because understanding the underlying tools is valuable.

But the IDE removes a lot of unnecessary friction while we're learning.

---

## 2. What is the JDK?

You'll often encounter:

```text
JDK
JRE
JVM
```

Don't confuse them.

For now, think of the relationship as:

```text
JDK
 ├── tools for developing JVM programs
 └── JVM/runtime components
```

The **JVM** is the environment that executes JVM bytecode.

The **JDK (Java Development Kit)** gives us the development tools and runtime needed to build JVM applications.

When creating a Kotlin project in IntelliJ, the project wizard asks you which JDK the project should use. IntelliJ can also download a JDK for you if one isn't already configured.

You don't need to install some separate mysterious "Kotlin machine."

---

# 3. Create our first Kotlin project

Open IntelliJ IDEA.

Choose:

```text
New Project
```

Then select:

```text
Kotlin
```

Give the project a name such as:

```text
KotlinLearning
```

For our initial learning project, use the **IntelliJ build system** rather than introducing Gradle immediately.

That's a deliberate choice.

Gradle is extremely important in real Kotlin development, and we will study it properly later. But right now, adding a build system would introduce another layer before you understand the language itself.

The current IntelliJ documentation describes the IntelliJ build system as a native builder that does not require downloading additional artifacts, while Gradle/Maven are available for more configurable projects.

For the JDK, choose an installed JDK or use **Download JDK** from the project wizard.

If you don't have a specific compatibility requirement, JetBrains' current documentation suggests Oracle OpenJDK as one option.

You can enable the sample code option and create the project.

---

# 4. What did IntelliJ actually create?

After the project opens, you'll see something resembling:

```text
KotlinLearning/
│
└── src/
    └── Main.kt
```

The exact structure can vary slightly depending on the project/build-system options.

Don't rush past this.

A **project** is a container for a body of related software.

The project isn't the program itself.

Inside the project we have source files.

For example:

```text
KotlinLearning
     │
     └── Main.kt
```

Later, a serious application might look more like:

```text
KotlinLearning
│
├── source files
├── tests
├── configuration
├── dependencies
├── resources
└── build information
```

So:

> **Project = the larger unit containing the things needed to develop and build an application.**

We'll return to this idea when we study Gradle and larger projects.

---

# 5. Open `Main.kt`

You may see something similar to:

```kotlin
fun main() {
    println("Hello World!")
}
```

This is our first real Kotlin source file.

Let's not add anything yet.

Look at:

```kotlin
fun main()
```

and remember our previous lesson:

- `fun` indicates a function declaration.
    
- `main` is the function's name.
    
- `()` means this function currently takes no parameters.
    
- `{ ... }` contains the function body.
    

The `main` function is the entry point of our application.

In other words, when we ask the environment to run the program, execution begins here.

---

# 6. Run it

You should see a green run icon near the `main` function.

Click it and choose the option to run `main`.

The output window should show:

```text
Hello World!
```

We've now completed the entire basic cycle:

```text
Write code
   ↓
Compile/build
   ↓
Run
   ↓
Observe output
```

This cycle is going to become extremely familiar.

---

# 7. Change the program

Replace the contents with:

```kotlin
fun main() {
    println("Hello, Kotlin!")
    println("I am learning programming.")
    println(10 + 20)
}
```

Run it.

You should see:

```text
Hello, Kotlin!
I am learning programming.
30
```

Notice something important.

These:

```kotlin
println("Hello, Kotlin!")
println("I am learning programming.")
```

give `println` a piece of text.

But:

```kotlin
println(10 + 20)
```

gives it an **expression**.

Kotlin evaluates:

```text
10 + 20
```

which produces:

```text
30
```

and then `println` prints that result.

So there are already two different ideas hiding here:

```text
values
expressions
```

We'll explore both extensively.

---

# 8. What actually happens when you press Run?

This is worth understanding.

When you press Run, IntelliJ doesn't somehow "send the text to the CPU."

Conceptually, something closer to this happens:

```text
Main.kt
   │
   │ Kotlin source code
   ▼
Kotlin compiler
   │
   │ compilation
   ▼
JVM bytecode
   │
   ▼
JVM
   │
   ▼
Program executes
   │
   ▼
Output
```

The details underneath are more complicated than this diagram, but this mental model is correct enough for now.

The Kotlin compiler translates Kotlin source into code for a selected target. For JVM development, that means JVM-compatible output. Kotlin's compiler can also target JavaScript and native platforms, which we'll study much later.

---

# 9. A source file is not the same thing as a running program

This distinction is easy to miss.

When you're looking at:

```kotlin
fun main() {
    println("Hello")
}
```

you're looking at **source code**.

Nothing has happened yet merely because the text exists.

The source code describes what the program should do.

Then the compiler translates it.

Then the resulting program is executed.

So:

```text
Source code
```

is a **description**.

Whereas:

```text
Running program
```

is an **execution of that description**.

This distinction becomes extremely useful later when we start talking about compilation errors, runtime errors and debugging.

---

# 10. What happens if the code is wrong?

Try this:

```kotlin
fun main() {
    println("Hello"
}
```

Notice the missing `)`.

IntelliJ should immediately complain.

This is a **compile-time problem**.

The compiler cannot successfully understand/compile the program.

Therefore, we cannot get to the normal execution stage.

Conceptually:

```text
Invalid source
     ↓
Compiler
     ↓
❌ compilation fails
     ↓
Program doesn't run normally
```

Now compare that with a program that compiles but produces an unexpected result.

For example:

```kotlin
fun main() {
    println(10 / 0)
}
```

This is a different category of problem.

The exact behavior and exception involved will become more interesting when we study types and exceptions, but the important conceptual distinction is:

```text
Compile time
    ↓
Can this source code be compiled?

Runtime
    ↓
What happens when the compiled program actually executes?
```

We'll keep coming back to that distinction.

---

# 11. Why does IntelliJ underline things before we run?

You may notice that IntelliJ sometimes tells you something is wrong while you're still typing.

That's because the IDE performs a lot of analysis of your source code.

For example, it can detect:

- syntax problems
    
- type mismatches
    
- unresolved names
    
- unused variables
    
- suspicious code
    
- possible mistakes
    

This is one of the biggest advantages of an IDE.

But here's an important principle:

> **The IDE is helping you understand your code; it is not the Kotlin language itself.**

Kotlin exists independently of IntelliJ.

You could write Kotlin using another editor and compile it with the command-line compiler.

The official Kotlin distribution includes a standalone compiler as well.

We'll explore that later so you understand what the IDE is doing for you rather than treating it as magic.

---

# 12. One more experiment

Write:

```kotlin
fun main() {
    println(10)
    println(20)
    println(30)
}
```

Now change it to:

```kotlin
fun main() {
    println(10 + 20 + 30)
}
```

Both programs print:

```text
60
```

But they're not doing the same thing internally.

The first program contains three separate calls to `println`.

The second contains one call to `println`, whose argument is an expression.

That's our first small taste of an important programming concept:

> **Different programs can produce the same output.**

Therefore:

```text
Output ≠ complete explanation of a program
```

Two implementations may behave the same for some inputs while having very different structures, performance, flexibility and correctness.

This becomes extremely important when we start writing larger programs.

---

# 13. Let's make the program slightly more interesting

Try:

```kotlin
fun main() {
    println("The calculation is:")
    println(10 + 20)
}
```

Now:

```kotlin
fun main() {
    println("The calculation is:")
    println(10 + 20)
    println(100 - 30)
    println(5 * 6)
}
```

We're manually writing every operation.

But imagine we eventually need to calculate:

```text
price × quantity
```

The values might change every time the program runs.

We don't want to rewrite the source code every time the price changes.

That leads us naturally to our next major concept:

**variables and values.**

---

# 14. One thing to notice before we move on

Take this:

```kotlin
println(10 + 20)
```

There are actually three distinct things involved:

```text
10
+
20
```

and then:

```text
10 + 20
```

is an expression.

The expression evaluates to a value:

```text
30
```

Then:

```text
println(...)
```

uses that value.

Conceptually:

```text
10       20
 \       /
  \     /
   10 + 20
       ↓
      30
       ↓
    println
       ↓
     output
```

This idea of **expressions producing values** is going to become one of the foundations of Kotlin.

---

# 15. Your first programming habit

From this point onward, I want you to stop treating the Run button as an oracle.

Before running a program, predict what it will do.

For example:

```kotlin
fun main() {
    println(5 + 3)
    println(5 * 3)
    println(20 - 7)
}
```

Predict:

```text
?
?
?
```

Then run it.

Then compare your prediction with reality.

The sequence should become:

```text
READ
  ↓
THINK
  ↓
PREDICT
  ↓
RUN
  ↓
COMPARE
  ↓
UNDERSTAND
```

That is much more valuable than simply experimenting randomly until the compiler accepts your code.

---

# 16. Before the next lesson

Make sure you can successfully create and run this:

```kotlin
fun main() {
    println("Hello, Kotlin!")
    println(10 + 20)
}
```

Then experiment freely.

Try changing:

```text
Hello, Kotlin!
```

to your own sentence.

Try changing:

```text
10 + 20
```

to different arithmetic expressions.

Try adding more `println` calls.

Try deliberately breaking the syntax and observing what IntelliJ tells you.

Don't worry about understanding every error yet.

We're learning the environment as well as the language.

---

## What you should understand now

You don't need to memorize the syntax of everything we've seen.

You should now have a reasonably solid mental picture of:

```text
IDE
  ↓
Project
  ↓
Source file
  ↓
Kotlin code
  ↓
Compiler
  ↓
Compiled program
  ↓
JVM
  ↓
Execution
  ↓
Output
```

And within the code:

```text
function
   ↓
main
   ↓
statements / expressions
   ↓
values
   ↓
output
```

The next lesson will begin the first genuinely important Kotlin building block:

# Variables, values, `val`, and `var`

We'll use them to answer questions such as:

> Where does a value live after we calculate it?

> How does a program remember information?

> What's the difference between a value and a variable?

> What exactly does `val` mean?

> Why does Kotlin distinguish `val` and `var`?

Those questions take us from "printing little expressions" into actual programming.