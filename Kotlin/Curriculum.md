
Absolutely. I’d structure this as a **complete Kotlin curriculum from zero programming knowledge to advanced, production-level Kotlin**, rather than as a collection of syntax tutorials.

The goal is that by the end, you should be able to look at an unfamiliar Kotlin codebase and understand it, design your own applications, use Kotlin idiomatically, debug problems, write tests, work with concurrency, and understand the architectural ideas behind real Kotlin projects.

I’d teach it in layers so that we **do not rush into Android, frameworks, or coroutines before the underlying language is genuinely understood**.

# Kotlin curriculum

## Phase 0 — Programming foundations

Before Kotlin itself, we establish the mental model necessary to program.

### Concepts

- What programming actually is
    
- Source code → compiler → executable/program
    
- Variables and memory at a conceptual level
    
- Expressions vs statements
    
- Values and types
    
- Control flow
    
- Functions
    
- Objects and state
    
- Data structures
    
- Algorithms at an introductory level
    
- Errors and debugging
    
- How programs are decomposed into smaller problems
    

### Kotlin setup

- Installing the JDK
    
- IntelliJ IDEA
    
- Kotlin compiler
    
- Kotlin project structure
    
- Running a Kotlin program
    
- REPL / scratch files
    
- Gradle at a basic level
    
- Reading compiler errors
    
- IDE navigation, autocomplete, refactoring and debugging
    

### Extent

Enough programming fundamentals that Kotlin syntax is not the thing you're struggling with.

---

# Phase 1 — Kotlin fundamentals

This is where you learn the actual language.

## 1. Basic syntax

- `fun main()`
    
- Comments
    
- Statements and expressions
    
- Semicolons and Kotlin's syntax philosophy
    
- Identifiers
    
- Keywords
    
- Naming conventions
    
- Code formatting
    

## 2. Variables

- `val`
    
- `var`
    
- Immutable vs mutable references
    
- Reassignment
    
- Type inference
    
- Explicit types
    
- Scope
    
- Local variables
    
- Constants
    
- `const val`
    

We'll spend significant time on **why `val` is generally preferred**, rather than merely memorizing the rule.

## 3. Types

Built-in types:

- `Int`
    
- `Long`
    
- `Short`
    
- `Byte`
    
- `Double`
    
- `Float`
    
- `Boolean`
    
- `Char`
    
- `String`
    

Then:

- Numeric conversions
    
- Type inference
    
- Type checking
    
- Type casting
    
- `is`
    
- `!is`
    
- `as`
    
- `as?`
    

We'll also discuss Kotlin's type system conceptually instead of treating types as syntax.

## 4. Strings

- String literals
    
- Escaping
    
- Raw strings
    
- String templates
    
- Multiline strings
    
- String operations
    
- Formatting
    
- Performance considerations
    

## 5. Operators

- Arithmetic
    
- Comparison
    
- Equality
    
- Logical operators
    
- Range operators
    
- Assignment
    
- `in`
    
- `!in`
    
- Operator precedence
    

---

# Phase 2 — Control flow

## Conditionals

- `if`
    
- `else`
    
- Nested conditions
    
- `if` as an expression
    

## `when`

From basic:

```kotlin
when (x) {
    1 -> ...
    2 -> ...
    else -> ...
}
```

to advanced usage:

- Multiple conditions
    
- Ranges
    
- Type checks
    
- Exhaustiveness
    
- `when` without an argument
    
- Using `when` as an expression
    

## Loops

- `for`
    
- `while`
    
- `do while`
    
- Ranges
    
- Progressions
    
- `step`
    
- `downTo`
    
- `until`
    

## Loop control

- `break`
    
- `continue`
    
- Labels
    
- Labeled returns
    

### Mini-projects

We'd build small programs such as:

- Number guessing game
    
- Calculator
    
- Unit converter
    
- Text analyzer
    
- Simple command-line menu system
    

---

# Phase 3 — Functions

This is a major phase because Kotlin becomes much more powerful once functions are properly understood.

## Core functions

- Parameters
    
- Return values
    
- Return types
    
- Expression-bodied functions
    
- Default arguments
    
- Named arguments
    
- `Unit`
    
- Local functions
    

## Advanced function concepts

- Function types
    
- Functions as values
    
- Higher-order functions
    
- Passing functions as arguments
    
- Returning functions
    
- Function references
    
- Lambdas
    
- Anonymous functions
    
- Closures
    

## Kotlin-specific function features

- Trailing lambdas
    
- Lambda parameter conventions
    
- `it`
    
- Destructuring
    
- Non-local returns
    
- `inline`
    
- `noinline`
    
- `crossinline`
    

We'll eventually connect this to the standard library so constructs like:

```kotlin
numbers.filter { it > 10 }
```

stop looking magical.

---

# Phase 4 — Null safety

This deserves an entire phase because Kotlin's null-safety model is one of its defining features.

## Concepts

- What `null` represents
    
- Nullable vs non-nullable types
    
- `String` vs `String?`
    
- Safe calls: `?.`
    
- Elvis operator: `?:`
    
- Explicit null checks
    
- Smart casts
    
- `let`
    
- `run`
    
- `also`
    
- `apply`
    
- `with`
    

## Dangerous escape hatch

- `!!`
    

We'll understand both **how it works and why excessive use is usually a design smell**.

## Deeper type-system ideas

- Platform types
    
- Java interoperability and nullability
    
- Nullable generics
    
- Generic nullable types
    
- Smart-cast limitations
    

### Exercises

We'll deliberately create buggy programs involving `null` and learn to diagnose the underlying type problem rather than just adding `!!`.

---

# Phase 5 — Collections

We'll go considerably beyond "here's how to create a list."

## Collection types

- `List`
    
- `Set`
    
- `Map`
    
- Mutable variants
    
- Read-only vs mutable collections
    

## Creating collections

- `listOf`
    
- `mutableListOf`
    
- `setOf`
    
- `mapOf`
    
- etc.
    

## Traversal

- `for`
    
- `forEach`
    
- Iterators
    

## Transformations

- `map`
    
- `mapNotNull`
    
- `filter`
    
- `filterNot`
    
- `flatMap`
    
- `flatten`
    
- `zip`
    
- `associate`
    
- `groupBy`
    
- `partition`
    

## Aggregation

- `sum`
    
- `average`
    
- `count`
    
- `reduce`
    
- `fold`
    
- `min`
    
- `max`
    

## Searching

- `find`
    
- `first`
    
- `firstOrNull`
    
- `any`
    
- `all`
    
- `none`
    
- `contains`
    

## Ordering

- `sorted`
    
- `sortedBy`
    
- `sortedWith`
    
- Comparators
    

## Sequences

- Why collections and sequences differ
    
- Lazy evaluation
    
- Intermediate vs terminal operations
    
- Performance implications
    
- When sequences help
    
- When they don't
    

### Project

A **command-line data-processing application** that reads, filters, transforms, groups and analyzes real data.

---

# Phase 6 — Object-oriented Kotlin

Now we move into object-oriented programming properly.

## Classes

- Class declaration
    
- Properties
    
- Constructors
    
- Primary constructors
    
- Secondary constructors
    
- `init`
    
- Property initialization
    
- Visibility
    

## Encapsulation

- `private`
    
- `protected`
    
- `internal`
    
- `public`
    
- Getters and setters
    
- Custom accessors
    

## Inheritance

- `open`
    
- `override`
    
- Abstract classes
    
- Method overriding
    
- Property overriding
    

## Interfaces

- Declaring interfaces
    
- Implementing interfaces
    
- Default implementations
    
- Multiple interfaces
    

## Object declarations

- `object`
    
- Singleton-like objects
    
- Companion objects
    

## Nested and inner classes

- Nested classes
    
- `inner`
    

## Data classes

- `data class`
    
- `copy`
    
- `componentN`
    
- Destructuring
    
- Equality
    
- `toString`
    

## Enum classes

- Basic enums
    
- Enum properties
    
- Enum methods
    

---

# Phase 7 — Kotlin's modeling features

This is where Kotlin starts becoming substantially more expressive than basic Java-style OOP.

## Sealed classes

- Sealed classes
    
- Sealed interfaces
    
- Modeling finite states
    
- Exhaustive `when`
    

For example:

```kotlin
sealed interface Result

data class Success(val data: String) : Result
data class Failure(val error: String) : Result
```

We'll use these ideas extensively in later projects.

## Value classes

- `@JvmInline`
    
- Type safety without unnecessary wrappers
    
- Domain modeling
    

## Type aliases

- `typealias`
    
- Appropriate vs inappropriate uses
    

## Delegation

- Interface delegation
    
- `by`
    
- Property delegation
    

## Delegated properties

- `lazy`
    
- `observable`
    
- Custom delegates
    

---

# Phase 8 — Generics

This will be taught deeply.

## Basics

- Generic classes
    
- Generic functions
    
- Type parameters
    
- Multiple type parameters
    

## Variance

- `out`
    
- `in`
    
- Invariance
    
- Covariance
    
- Contravariance
    
- Why variance exists
    

This is one of the areas where many Kotlin tutorials become shallow. We'll work through the underlying type relationships.

## Advanced generics

- Generic constraints
    
- Multiple constraints
    
- Reified type parameters
    
- Generic extension functions
    
- Type erasure
    
- JVM implications
    

---

# Phase 9 — Extensions

## Extension functions

- Syntax
    
- Dispatch
    
- Resolution rules
    
- Nullable receivers
    
- Generic receivers
    

## Extension properties

- When they are useful
    
- Their limitations
    

## Design implications

- Extensions vs member functions
    
- API design
    
- Avoiding extension abuse
    

We'll use extensions heavily, but we'll also learn when **not** to use them.

---

# Phase 10 — Kotlin idioms and expressive syntax

At this stage, you'll know Kotlin, but we start learning **Kotlin as Kotlin**, rather than "Java syntax with Kotlin punctuation."

Topics include:

- Expression-oriented programming
    
- Smart casts
    
- Destructuring
    
- Scope functions
    
- `takeIf` / `takeUnless`
    
- `require`
    
- `check`
    
- `error`
    
- Ranges
    
- Contracts conceptually
    
- Operator overloading
    
- Infix functions
    
- Local returns
    
- Labels
    
- `Nothing`
    

We'll learn how to distinguish:

> concise Kotlin

from

> clever Kotlin that nobody wants to maintain.

That's an important distinction in real projects.

---

# Phase 11 — Exceptions and error handling

## Exceptions

- `try`
    
- `catch`
    
- `finally`
    
- Multiple catches
    
- `try` as an expression
    

## Designing errors

- Exceptions vs returned errors
    
- Validation errors
    
- Domain errors
    
- Recoverable vs unrecoverable problems
    

## Kotlin's approach

- `Result`
    
- `runCatching`
    
- Sealed result types
    
- Error propagation
    

We'll compare several approaches rather than pretending there is one universal answer.

---

# Phase 12 — Standard library mastery

Rather than simply memorizing APIs, we'll learn how to navigate and reason about the standard library.

Areas include:

- Strings
    
- Collections
    
- Sequences
    
- Ranges
    
- I/O basics
    
- Regex
    
- Files
    
- Time/date APIs
    
- Utility functions
    
- Scope functions
    
- `Result`
    

You'll learn to answer:

> "I need to transform this data structure. How do I discover the appropriate Kotlin API?"

That skill matters much more than memorizing 200 functions.

---

# Phase 13 — File I/O and command-line applications

We'll build serious console applications before touching Android.

Topics:

- Reading files
    
- Writing files
    
- Paths
    
- Directories
    
- Buffered I/O
    
- CSV/JSON concepts
    
- Command-line arguments
    
- Environment variables
    
- Configuration
    
- Logging basics
    

### Project

A **real CLI application** with:

- multiple commands
    
- configuration
    
- persistence
    
- validation
    
- error handling
    
- tests
    
- packaging
    

---

# Phase 14 — Gradle and Kotlin project structure

This is essential for real Kotlin development.

## Gradle fundamentals

- What Gradle does
    
- Projects
    
- Modules
    
- Dependencies
    
- Tasks
    
- Plugins
    
- Build lifecycle
    

## Kotlin projects

- `build.gradle.kts`
    
- `settings.gradle.kts`
    
- Source sets
    
- Dependency management
    
- Version catalogs
    
- Multi-module projects
    

## Build concepts

- Debug vs release
    
- Dependency scopes
    
- Reproducible builds
    
- Packaging
    

You won't need to become a Gradle specialist, but you should be comfortable enough to maintain and troubleshoot real projects.

---

# Phase 15 — Testing

We don't wait until the end to introduce testing, but here we'll learn it systematically.

## Testing fundamentals

- Unit testing
    
- Test cases
    
- Assertions
    
- Test organization
    
- Test isolation
    
- Edge cases
    

## Kotlin testing

- JUnit
    
- Kotlin-friendly assertions
    
- Parameterized tests
    
- Test fixtures
    
- Mocking concepts
    

## What makes a good test?

- Behavior vs implementation
    
- Deterministic tests
    
- Readability
    
- Test naming
    
- Testing failures
    
- Regression tests
    

## Test-driven development

We'll experiment with TDD rather than treating it as a religion.

---

# Phase 16 — Java interoperability

This is crucial because Kotlin exists heavily inside the JVM ecosystem.

## Java from Kotlin

- Calling Java classes
    
- Java collections
    
- Java APIs
    
- Checked exceptions
    
- Static members
    
- Getters/setters
    

## Kotlin from Java

- JVM signatures
    
- `@JvmStatic`
    
- `@JvmOverloads`
    
- `@JvmField`
    
- File facades
    
- Naming generated bytecode
    

## Interop problems

- Platform types
    
- Nullability
    
- Generic variance
    
- SAM conversions
    
- Java annotations
    
- `Optional`
    

By the end, reading mixed Kotlin/Java code should feel normal.

---

# Phase 17 — JVM fundamentals

Not a full JVM course, but enough to understand what Kotlin ultimately runs on.

We'll cover:

- JVM overview
    
- Bytecode conceptually
    
- Heap vs stack conceptually
    
- Objects and references
    
- Garbage collection
    
- Boxing/unboxing
    
- Primitive types on the JVM
    
- Generics and type erasure
    
- Class loading basics
    
- Performance implications of abstraction
    

The goal isn't to become a JVM engineer.

The goal is to understand why seemingly innocent Kotlin code can behave the way it does.

---

# Phase 18 — Coroutines

This is another major phase.

We won't teach coroutines as "replace threads with `suspend`."

We'll build the conceptual model first.

## Foundations

- Processes
    
- Threads
    
- Concurrency
    
- Parallelism
    
- Blocking
    
- Non-blocking
    

Then:

- `suspend`
    
- Coroutine builders
    
- `launch`
    
- `async`
    
- `await`
    
- `runBlocking`
    

## Coroutine context

- CoroutineContext
    
- Dispatchers
    
- Jobs
    
- Structured concurrency
    

## Scope

- CoroutineScope
    
- Scope ownership
    
- Cancellation
    

## Cancellation

- Cooperative cancellation
    
- `isActive`
    
- `ensureActive`
    
- `yield`
    
- Non-cancellable sections
    

## Exception handling

- Coroutine failures
    
- `CoroutineExceptionHandler`
    
- `supervisorScope`
    
- `SupervisorJob`
    

## Advanced

- Coroutine internals conceptually
    
- Suspending functions
    
- Sequential vs concurrent coroutine execution
    
- Shared mutable state
    
- Mutex
    
- Actors conceptually
    
- Channels
    

---

# Phase 19 — Flow and reactive programming

Then we'll move from individual asynchronous operations to streams of values.

## Flow basics

- `Flow`
    
- `flow`
    
- `collect`
    
- Cold streams
    

## Operators

- `map`
    
- `filter`
    
- `transform`
    
- `combine`
    
- `zip`
    
- `flatMap`
    
- `debounce`
    
- `distinctUntilChanged`
    

## Lifecycle

- Cancellation
    
- Exceptions
    
- Completion
    
- `catch`
    
- `onCompletion`
    

## Hot streams

- `StateFlow`
    
- `SharedFlow`
    

## Architecture

- State vs events
    
- State holders
    
- Backpressure concepts
    
- UI/event pipelines
    

---

# Phase 20 — Concurrency and thread safety

We'll go deeper than coroutines.

Topics:

- Race conditions
    
- Atomicity
    
- Visibility
    
- Shared mutable state
    
- Immutability
    
- Synchronization
    
- Locks
    
- Mutex
    
- Atomic variables
    
- Thread confinement
    
- Safe publication
    

We'll build concurrent programs that intentionally contain bugs and then fix them.

---

# Phase 21 — Serialization and networking

Before Android, we should know how applications communicate with external systems.

Topics:

- HTTP fundamentals
    
- REST APIs
    
- Requests/responses
    
- JSON
    
- Serialization/deserialization
    
- Kotlin serialization
    
- Network errors
    
- Timeouts
    
- Retries
    
- Authentication concepts
    

### Project

A Kotlin application consuming a real HTTP API, caching results, handling errors and exposing a clean domain model.

---

# Phase 22 — Architecture and software design

This is where we move from:

> "I know Kotlin."

to:

> "I can build something maintainable with Kotlin."

Topics:

- Separation of concerns
    
- Coupling
    
- Cohesion
    
- Dependency inversion
    
- Composition over inheritance
    
- Interfaces
    
- Dependency injection concepts
    
- Repository pattern
    
- Service layer
    
- Domain modeling
    
- State management
    
- Immutability
    
- Functional vs object-oriented design
    

Then:

- Clean architecture concepts
    
- Hexagonal architecture concepts
    
- Layer boundaries
    
- Module boundaries
    

We will **not blindly apply architectures because they're fashionable**. We'll examine why they exist and when they introduce unnecessary complexity.

---

# Phase 23 — Functional programming in Kotlin

Kotlin isn't a pure functional language, but it supports many functional techniques.

Topics:

- Pure functions
    
- Referential transparency
    
- Immutability
    
- Higher-order functions
    
- Function composition
    
- Side effects
    
- Persistent data concepts
    
- Algebraic-style modeling
    
- `fold`
    
- `map`
    
- `flatMap`
    

We'll compare functional and object-oriented approaches and see where each fits naturally.

---

# Phase 24 — DSLs and advanced Kotlin

Now we reach genuinely advanced Kotlin.

## DSL construction

- Lambdas with receivers
    
- Type-safe builders
    
- Receivers
    
- Scope control
    
- DSL design
    

## Advanced language features

- `inline`
    
- `noinline`
    
- `crossinline`
    
- `reified`
    
- Delegation
    
- Contracts
    
- Context-related language features where applicable
    
- Compiler behavior
    
- Metadata concepts
    

### Projects

Build a small domain-specific language, such as:

```kotlin
workflow {
    step("download")
    step("transform")
    step("upload")
}
```

Then study how such APIs are actually designed.

---

# Phase 25 — Kotlin Multiplatform

Only after the core language is solid.

We'll examine:

- Kotlin Multiplatform concept
    
- Shared code
    
- Platform-specific code
    
- `expect` / `actual`
    
- Common source sets
    
- JVM / Android / iOS concepts
    
- Dependency constraints
    
- Architecture of shared applications
    

The focus will be understanding the model rather than memorizing framework APIs.

---

# Phase 26 — Android with Kotlin

This becomes a **major specialization**, not something mixed into the beginner lessons.

## Android foundations

- Android Studio
    
- Project structure
    
- Application lifecycle
    
- Activities
    
- Intents
    
- Resources
    
- Configuration
    
- Permissions
    

## Modern UI

- Jetpack Compose
    
- Composable functions
    
- State
    
- Recomposition
    
- Composition
    
- Modifiers
    
- Layout
    
- Lists
    
- Navigation
    
- Theming
    

## State architecture

- ViewModel
    
- StateFlow
    
- UI state
    
- Events
    
- Lifecycle-aware collection
    

## Persistence

- Room
    
- DataStore
    
- Local caching
    

## Networking

- HTTP
    
- Retrofit concepts
    
- Serialization
    
- Repository layer
    

## Dependency injection

- DI concepts
    
- Hilt/Dagger concepts
    

## Android concurrency

- Coroutines
    
- Flow
    
- Lifecycle scopes
    
- structured concurrency in Android
    

## Testing

- ViewModel tests
    
- Repository tests
    
- Compose testing
    
- Integration testing
    

### Major Android project

A production-style application involving:

**UI → ViewModel → domain logic → repository → database/network**

with loading, error, caching, navigation, tests and offline behavior.

---

# Phase 27 — Production Kotlin

This is where we stop behaving like tutorial programmers.

Topics:

- Codebase organization
    
- API design
    
- Module boundaries
    
- Dependency management
    
- Logging
    
- Configuration
    
- Environment separation
    
- Error reporting
    
- Performance
    
- Memory usage
    
- Concurrency issues
    
- Security basics
    
- Code review
    
- Refactoring
    
- Backward compatibility
    
- Semantic versioning
    

## Performance

We'll investigate:

- Allocation
    
- Boxing
    
- Collections
    
- Sequences
    
- Inline functions
    
- Coroutine overhead
    
- String operations
    
- Algorithmic complexity
    
- JVM profiling concepts
    

---

# Phase 28 — Reading real-world Kotlin

A surprisingly important skill.

We'll take unfamiliar Kotlin code and learn to decode:

```kotlin
repository
    .getUsers()
    .filter { it.isActive }
    .map(::toUserModel)
    .catch { emit(emptyList()) }
    .flowOn(Dispatchers.IO)
```

Instead of seeing it as a wall of Kotlin syntax, you'll learn to mentally expand it into:

1. What type is flowing through the pipeline?
    
2. What is executed immediately?
    
3. What is lazy?
    
4. What can suspend?
    
5. Where can exceptions occur?
    
6. Which thread/dispatcher matters?
    
7. Who owns the lifecycle?
    
8. What state is mutable?
    
9. What happens on cancellation?
    

That's the level of understanding I want you to reach.

---

# Phase 29 — Large capstone

The final project combines everything.

We would build a **substantial Kotlin application** with:

- multi-module Gradle structure
    
- domain models
    
- generics
    
- sealed types
    
- collections and sequences
    
- coroutines
    
- Flow
    
- networking
    
- serialization
    
- persistence
    
- dependency injection
    
- error handling
    
- logging
    
- testing
    
- concurrency
    
- clean architecture principles
    
- documentation
    
- performance considerations
    

Depending on the direction you eventually choose, this could become:

**Backend/server application**, **Android application**, **multiplatform application**, or **general JVM application**.

---

# What "learn Kotlin in detail" means here

I would divide mastery into roughly four levels.

|Level|What you should be able to do|
|---|---|
|**Beginner**|Write small Kotlin programs without copying syntax|
|**Intermediate**|Build complete applications using collections, OOP, generics, extensions and standard library features|
|**Advanced**|Reason about type systems, coroutines, Flow, concurrency, JVM behavior and API design|
|**Production**|Design, test, debug, refactor and maintain substantial Kotlin codebases|

I would **not** consider you finished merely because you know every keyword.

The real endpoint is:

> You can encounter unfamiliar Kotlin code and derive what it does from first principles.

---

# How I'd teach each topic

Rather than doing:

> "Here is `map()`. Memorize this."

we'll use a repeated progression:

### 1. Mental model

Understand what the feature is solving.

### 2. Minimal syntax

Learn the smallest possible example.

### 3. Variations

Explore the important forms.

### 4. Common mistakes

See how beginners misuse it.

### 5. Internals

Understand what Kotlin/JVM is doing when it matters.

### 6. Practice

Solve progressively harder problems.

### 7. Real code

Use the concept in a project.

### 8. Refactoring

Take working-but-bad Kotlin and improve it.

That last step is particularly important because **writing code and writing good Kotlin are different skills**.

---

# The project progression

Rather than 100 disconnected exercises, I'd use projects that grow with your ability:

**Stage 1**

```text
Calculator
Number guessing game
Text analyzer
```

**Stage 2**

```text
CLI task manager
Expense tracker
File-based contact manager
```

**Stage 3**

```text
Data processing application
HTTP API client
JSON-backed application
```

**Stage 4**

```text
Concurrent application
Coroutine/Flow project
Multi-module Kotlin application
```

**Stage 5**

```text
Production-style capstone
```

And throughout the curriculum, we'll have smaller exercises specifically designed to expose misunderstandings.

---

# What we will deliberately NOT do

There are several common ways Kotlin courses become misleading.

We won't start with Android.

We won't teach Kotlin as "Java but shorter."

We won't throw coroutines at you before you understand functions, types and concurrency.

We won't make you memorize standard-library functions without understanding their abstractions.

We won't use `!!`, mutable global state, giant classes and overly clever one-liners and then call that idiomatic Kotlin.

We won't hide the JVM underneath a magical abstraction layer.

And we won't confuse **knowing Kotlin syntax** with **knowing how to engineer software**.

---

# Final curriculum map

The overall progression is essentially:

```text
Programming Fundamentals
        ↓
Kotlin Syntax & Types
        ↓
Control Flow
        ↓
Functions & Lambdas
        ↓
Null Safety
        ↓
Collections
        ↓
OOP
        ↓
Sealed/Data/Value Types
        ↓
Generics
        ↓
Extensions & Kotlin Idioms
        ↓
Error Handling
        ↓
Standard Library
        ↓
File/CLI Applications
        ↓
Gradle
        ↓
Testing
        ↓
Java Interoperability
        ↓
JVM Fundamentals
        ↓
Coroutines
        ↓
Flow
        ↓
Concurrency
        ↓
Networking & Serialization
        ↓
Architecture & Design
        ↓
Functional Programming
        ↓
Advanced Kotlin / DSLs
        ↓
Kotlin Multiplatform
        ↓
Android / Backend specialization
        ↓
Production Engineering
        ↓
Capstone
```

### The important distinction

I would treat **Core Kotlin** as the main course:

**Phases 0–24**

and everything after that as applying Kotlin to larger ecosystems:

**Multiplatform → Android/backend → production engineering.**

That keeps the language itself from getting buried under frameworks.

## What you'll ultimately know

By the end of the core curriculum, you should be comfortable with essentially all of the Kotlin concepts that matter in serious day-to-day development:

**syntax, types, null safety, functions, lambdas, collections, generics, OOP, sealed types, extensions, delegation, scope functions, exceptions, standard library, DSLs, Java interop, JVM behavior, coroutines, Flow, concurrency, testing, Gradle, serialization, networking, and software design.**

And importantly, we'll continuously distinguish **"Kotlin feature"**, **"programming concept"**, and **"software-engineering practice"**, because they're related but not the same thing.

### One thing I'd change from a conventional course

I wouldn't make this a linear textbook where you finish Chapter 1 before ever seeing Chapter 20.

We'll occasionally **preview advanced ideas early**, then return to them after you've acquired the prerequisite knowledge. For example, you might see a coroutine-based program early, but we won't properly study coroutines until you've built the conceptual foundation needed to understand what it is actually doing.

That produces a much stronger understanding than simply marching through syntax.

When we start the actual lessons, I'd begin with **"What is a program, what is Kotlin, what does the compiler do, and what exactly happens when `main()` runs?"** Then we'll write the first few Kotlin programs and build the mental model from there.