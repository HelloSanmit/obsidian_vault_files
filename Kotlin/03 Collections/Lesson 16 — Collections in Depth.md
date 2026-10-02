
# Lesson 16 — Collections in Depth: `List`, `Set`, `Map`, Mutability, and Iteration

We've already used collections:

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)

val evenNumbers = numbers.filter { it % 2 == 0 }
```

At that level, you can _use_ collections.

Now I want you to understand them well enough that when you see:

```kotlin
List<User>
MutableMap<String, Int>
Set<String>
```

you can immediately reason about what kind of data structure it is, what operations make sense, and what assumptions you can safely make.

This lesson is therefore less about learning twenty more functions and more about understanding **what these collections actually represent**.

---

# 1. First: what problem are collections solving?

Suppose we need to store one student's name:

```kotlin
val name = "Alice"
```

Easy.

Now suppose we need:

```text
Alice
Bob
Charlie
David
Eve
...
```

We need a single abstraction representing **many values**.

That's what a collection gives us.

But the important question is:

> **How are those values related?**

That's what determines whether we want a `List`, `Set`, or `Map`.

---

# 2. `List`: a sequence of elements

A `List` is ordered.

```kotlin
val numbers = listOf(10, 20, 30, 40)
```

Conceptually:

```text
index    value

  0       10
  1       20
  2       30
  3       40
```

So a list gives us:

- a sequence,
    
- positions,
    
- duplicates,
    
- a stable notion of "first", "second", etc.
    

For example:

```kotlin
val names = listOf("Alice", "Bob", "Alice")
```

is completely valid.

There are two `Alice`s.

That makes sense because the list is representing **three positions containing three values**.

---

# 3. Indexing

You can access an element using:

```kotlin
val numbers = listOf(10, 20, 30)

println(numbers[0])
```

Output:

```text
10
```

The index starts at `0`.

So:

```text
numbers[0] → first element
numbers[1] → second element
numbers[2] → third element
```

You can also use:

```kotlin
numbers.first()
numbers.last()
```

which often communicate intent better than indexing.

For example:

```kotlin
println(numbers.first())
println(numbers.last())
```

---

# 4. Indexing has a cost and a danger

Consider:

```kotlin
val numbers = listOf(10, 20, 30)

println(numbers[100])
```

There's no element at index `100`.

That causes an exception at runtime.

This is one reason Kotlin provides safer-looking operations such as:

```kotlin
numbers.getOrNull(100)
```

which returns:

```text
null
```

rather than failing with an invalid-index exception.

This connects directly to our null-safety lesson:

```text
get(index)
   ↓
assumes index is valid

getOrNull(index)
   ↓
acknowledges that the element may not exist
```

Again, notice how Kotlin often gives you an API whose return type communicates uncertainty.

---

# 5. List size

You can ask:

```kotlin
val numbers = listOf(10, 20, 30)

println(numbers.size)
```

Result:

```text
3
```

This gives us a useful distinction:

```text
last index = size - 1
```

So if:

```text
size = 3
```

the valid indexes are:

```text
0, 1, 2
```

This is why writing:

```kotlin
numbers[numbers.size]
```

is wrong.

The last valid index is:

```kotlin
numbers.size - 1
```

You can also use:

```kotlin
numbers.lastIndex
```

which communicates that directly.

---

# 6. Don't manually manage indexes unless you actually need them

Suppose:

```kotlin
val names = listOf("Alice", "Bob", "Charlie")
```

If all you want is the names:

```kotlin
for (name in names) {
    println(name)
}
```

is usually better than:

```kotlin
for (i in 0 until names.size) {
    println(names[i])
}
```

Both work.

But the first says:

> "Give me each name."

The second says:

> "Give me each index, then I'll use the index to get the name."

If the index isn't relevant to the problem, don't introduce it unnecessarily.

This is a recurring programming principle:

> **Use the abstraction that matches the information you actually need.**

---

# 7. What if you need both the index and the value?

Then the index _is_ relevant.

Kotlin provides:

```kotlin
for ((index, name) in names.withIndex()) {
    println("$index: $name")
}
```

Output:

```text
0: Alice
1: Bob
2: Charlie
```

This is a nice example of a principle we'll see repeatedly:

> Don't throw away information you actually need, but don't introduce information you don't need.

---

# 8. `List` vs `MutableList`

This distinction is important enough to revisit.

```kotlin
val numbers = listOf(1, 2, 3)
```

gives you a read-only `List`.

Whereas:

```kotlin
val numbers = mutableListOf(1, 2, 3)
```

gives you a `MutableList`.

Now:

```kotlin
numbers.add(4)
```

is allowed for the mutable one.

You can also:

```kotlin
numbers.remove(2)
numbers[0] = 100
```

depending on the current contents.

---

# 9. `val` and mutable collections

This is a common source of confusion.

```kotlin
val numbers = mutableListOf(1, 2, 3)
```

Some beginners see `val` and think:

> "The list cannot change."

Not true.

`val` protects the **reference from reassignment**.

The collection itself is mutable.

So:

```kotlin
numbers.add(4)
```

is valid.

But:

```kotlin
numbers = mutableListOf(7, 8, 9)
```

isn't.

Conceptually:

```text
numbers
   │
   ▼
[1, 2, 3]

numbers.add(4)

   │
   ▼

[1, 2, 3, 4]
```

The variable still refers to the same list object.

Its contents changed.

This is one of the first places where understanding **references and objects** will eventually become important.

---

# 10. Read-only doesn't necessarily mean immutable

This is subtle.

Suppose:

```kotlin
val mutable = mutableListOf(1, 2, 3)
val readOnly: List<Int> = mutable
```

Now:

```kotlin
readOnly.add(4)
```

is not allowed.

But:

```kotlin
mutable.add(4)
```

is allowed.

Then:

```kotlin
println(readOnly)
```

will reflect the updated contents.

So `List` here is acting as a **read-only view/interface**, not as proof that nobody can modify the underlying collection.

This is why I'd like you to keep these concepts separate:

```text
read-only access
```

and:

```text
immutable data
```

They are related, but not identical.

Later we'll discuss genuinely immutable/persistent data structures.

---

# 11. `Set`: when identity is about membership, not position

Now consider this:

```kotlin
val usernames = setOf(
    "alice",
    "bob",
    "charlie"
)
```

What does the set conceptually represent?

Not:

```text
position 0 = alice
position 1 = bob
...
```

but:

> These values belong to a collection of unique values.

So:

```kotlin
val numbers = setOf(1, 2, 3, 2, 1)
```

represents the unique elements:

```text
1
2
3
```

The important property is uniqueness.

---

# 12. Why would a set be useful?

Suppose you're tracking registered usernames.

You don't want:

```text
alice
bob
alice
alice
```

as four separate registrations.

You care whether:

```text
"alice" is already present
```

So:

```kotlin
val registeredUsers = mutableSetOf<String>()

registeredUsers.add("alice")
registeredUsers.add("bob")
registeredUsers.add("alice")
```

The set still contains only:

```text
alice
bob
```

That makes the data structure itself enforce part of the rule.

This is a powerful idea:

> **Choose data structures that naturally enforce the properties your data needs.**

---

# 13. `add` on a set can tell you whether insertion happened

Suppose:

```kotlin
val users = mutableSetOf("alice", "bob")
```

Then:

```kotlin
println(users.add("charlie"))
```

returns:

```text
true
```

because `"charlie"` was newly inserted.

But:

```kotlin
println(users.add("alice"))
```

returns:

```text
false
```

because `"alice"` was already there.

This is a good reminder that mutation functions can themselves return useful information.

A function isn't necessarily:

> do something and return nothing.

It can mutate state **and** return a result describing what happened.

---

# 14. `Set` is useful for membership checks

For example:

```kotlin
val blockedWords = setOf("spam", "scam", "fraud")

val word = "spam"

if (word in blockedWords) {
    println("Blocked")
}
```

The expression:

```kotlin
word in blockedWords
```

produces a Boolean.

That's exactly the question a set is designed around:

> Is this value a member of the set?

---

# 15. `Map`: relationships between keys and values

Now let's think differently.

Suppose we have:

```text
Alice → 25
Bob → 30
Charlie → 21
```

A `List` isn't the best abstraction.

We have a relationship:

```text
key → value
```

That's a `Map`.

```kotlin
val ages = mapOf(
    "Alice" to 25,
    "Bob" to 30,
    "Charlie" to 21
)
```

Now:

```kotlin
println(ages["Bob"])
```

gives:

```text
30
```

The key is:

```text
"Bob"
```

and the associated value is:

```text
30
```

---

# 16. Keys are unique

A map can't meaningfully have two independent values for exactly the same key.

Consider:

```kotlin
val ages = mapOf(
    "Alice" to 25,
    "Alice" to 30
)
```

The duplicate key creates a collision, and the resulting map doesn't contain two separate `"Alice"` entries. In a map, keys identify the associated values.

This is exactly why the key must be something you can use to uniquely identify an entry.

---

# 17. Map lookups can return `null`

Suppose:

```kotlin
val ages = mapOf(
    "Alice" to 25,
    "Bob" to 30
)
```

Then:

```kotlin
val age = ages["Charlie"]
```

The result can be null.

Why?

Because `"Charlie"` may not be a key.

So the type is effectively:

```text
Int?
```

not just:

```text
Int
```

This is a fantastic example of the language making absence explicit.

You could write:

```kotlin
val age = ages["Charlie"] ?: 0
```

but whether `0` is a sensible fallback depends on the problem.

---

# 18. Don't confuse missing keys with keys whose value is null

Here's a more subtle case.

A map can itself have nullable values:

```kotlin
val users = mapOf(
    "Alice" to 25,
    "Bob" to null
)
```

Now what does:

```kotlin
users["Bob"]
```

mean?

The result is `null`.

But that could mean:

1. `"Bob"` exists and his value is null, or
    
2. `"Bob"` doesn't exist.
    

This illustrates an important general programming problem:

> Sometimes one representation isn't enough to distinguish two different states.

We'll eventually learn stronger modeling techniques for these situations.

For now, you can check:

```kotlin
users.containsKey("Bob")
```

to distinguish the cases.

---

# 19. Maps can be mutable too

```kotlin
val ages = mutableMapOf(
    "Alice" to 25,
    "Bob" to 30
)
```

Now you can do:

```kotlin
ages["Charlie"] = 21
```

or:

```kotlin
ages["Alice"] = 26
```

So a mutable map supports both:

```text
adding entries
```

and:

```text
changing existing values
```

You can also remove:

```kotlin
ages.remove("Bob")
```

---

# 20. A useful comparison

Think of the three structures like this.

### List

```text
[ Alice, Bob, Charlie ]
```

Question:

> What element is at position 1?

### Set

```text
{ Alice, Bob, Charlie }
```

Question:

> Is Alice a member?

### Map

```text
Alice   → 25
Bob     → 30
Charlie → 21
```

Question:

> What value is associated with Alice?

This is a far better mental model than simply memorizing:

```text
List = ordered
Set = unique
Map = key-value
```

because it connects the data structure to the **question the program needs to answer**.

---

# 21. Iteration over lists

You've already seen:

```kotlin
val numbers = listOf(10, 20, 30)

for (number in numbers) {
    println(number)
}
```

This is direct iteration.

But you can also use `forEach`:

```kotlin
numbers.forEach { number ->
    println(number)
}
```

These are conceptually similar.

The question is whether the lambda form actually improves readability.

Sometimes:

```kotlin
for (number in numbers) {
    println(number)
}
```

is clearer.

Don't let the fact that something is "functional Kotlin" convince you that it must always be better.

---

# 22. `map`: transform one collection into another

We've used:

```kotlin
val doubled = numbers.map { it * 2 }
```

Let's understand the guarantee.

Suppose:

```text
input:
[1, 2, 3, 4]
```

and:

```kotlin
{ it * 2 }
```

is applied.

Result:

```text
[2, 4, 6, 8]
```

Every input element produces one output element.

So if the original collection has:

```text
N elements
```

the normal `map` result has:

```text
N elements
```

That gives you an important invariant.

---

# 23. `filter`: select without changing the elements

Suppose:

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)
```

and:

```kotlin
val result = numbers.filter { it % 2 == 0 }
```

The lambda returns:

```text
Boolean
```

It's essentially answering:

> Keep this element?

So:

```text
1 → no
2 → yes
3 → no
4 → yes
5 → no
```

Result:

```text
[2, 4]
```

Unlike `map`, the elements themselves aren't transformed.

We're selecting a subset.

---

# 24. `map` and `filter` can be composed

For example:

```kotlin
val result = numbers
    .filter { it > 5 }
    .map { it * 10 }
```

Read it as:

> Keep values greater than 5, then multiply each remaining value by 10.

Suppose:

```text
[1, 3, 7, 10]
```

First:

```text
filter
↓
[7, 10]
```

Then:

```text
map
↓
[70, 100]
```

This is a pipeline.

And pipelines are essentially **data flowing through transformations**.

That way of thinking will become very powerful later with sequences and Flow.

---

# 25. `filter` can change the size; `map` normally doesn't

For:

```kotlin
numbers.map { ... }
```

you usually get:

```text
same number of elements
```

For:

```kotlin
numbers.filter { ... }
```

you might get:

```text
0 elements
1 element
half the elements
all elements
...
```

This distinction sounds trivial, but it's incredibly useful when mentally tracing pipelines.

---

# 26. Searching: `find`

Suppose:

```kotlin
val users = listOf("Alice", "Bob", "Charlie", "David")
```

and:

```kotlin
val result = users.find { it.startsWith("C") }
```

The result is:

```text
"Charlie"
```

But the result type is nullable:

```text
String?
```

because there may be no matching user.

For example:

```kotlin
val result = users.find { it.startsWith("Z") }
```

gives:

```text
null
```

Now our null-safety knowledge becomes useful again.

---

# 27. `first` vs `firstOrNull`

These look similar but express different assumptions.

```kotlin
numbers.first()
```

means:

> I expect there to be a first element.

If the collection is empty, that assumption fails and an exception occurs.

Whereas:

```kotlin
numbers.firstOrNull()
```

means:

> There may not be a first element.

Then the result can be:

```text
T?
```

This is a good example of API design communicating **how much certainty you have**.

Compare:

```text
first()
→ I'm assuming one exists

firstOrNull()
→ I'm acknowledging that one might not exist
```

That's a useful pattern to recognize throughout Kotlin's standard library.

---

# 28. `single` is stricter still

Suppose:

```kotlin
val users = listOf("Alice")
```

Then:

```kotlin
users.single()
```

works because there is exactly one element.

But if:

```text
[]
```

or:

```text
[Alice, Bob]
```

the assumption fails.

You can also write:

```kotlin
singleOrNull()
```

if zero or multiple matches should be treated as "no unique result."

This is a good example of how API names encode expectations.

```text
first
→ one or more is okay

single
→ exactly one is required
```

We'll encounter similar distinctions with databases, HTTP results, and domain logic later.

---

# 29. Aggregating data

Suppose:

```kotlin
val numbers = listOf(10, 20, 30)
```

We can ask:

```kotlin
numbers.sum()
```

→ `60`

```kotlin
numbers.average()
```

→ `20.0`

```kotlin
numbers.count()
```

→ `3`

We can also combine this with filtering:

```kotlin
val total = numbers
    .filter { it >= 20 }
    .sum()
```

Result:

```text
50
```

Again:

```text
data
 ↓
select
 ↓
aggregate
```

---

# 30. `fold`: the general accumulator

Let's revisit this because it reveals something deeper.

```kotlin
val numbers = listOf(1, 2, 3, 4)

val result = numbers.fold(0) { accumulator, number ->
    accumulator + number
}
```

Trace it:

```text
initial accumulator = 0

0 + 1 = 1
1 + 2 = 3
3 + 3 = 6
6 + 4 = 10
```

Final result:

```text
10
```

But `fold` isn't specifically "sum."

You can use it for many accumulation problems.

For multiplication:

```kotlin
val product = numbers.fold(1) { acc, number ->
    acc * number
}
```

Result:

```text
24
```

Now notice what's happening conceptually:

```text
many values
    ↓
repeatedly combine them
    ↓
one final result
```

This is a powerful abstraction.

---

# 31. The initial value matters

Suppose you use:

```kotlin
numbers.fold(0) { ... }
```

The accumulator begins at `0`.

For addition, that's sensible.

For multiplication:

```kotlin
fold(1)
```

is more natural because `1` doesn't alter the first multiplication.

This is connected to a deeper mathematical concept: **identity values**.

For addition:

```text
0 + x = x
```

For multiplication:

```text
1 × x = x
```

We won't go into abstract algebra yet, but this is the kind of structure that `fold` captures.

---

# 32. `reduce` is related, but different

You can also see:

```kotlin
val sum = numbers.reduce { acc, number ->
    acc + number
}
```

The important difference at this stage is:

```text
fold
→ you provide the initial accumulator

reduce
→ the collection supplies the initial accumulator
```

That makes empty collections particularly important.

For example, how could `reduce` determine an initial value if there are no elements?

It can't.

That's why `reduce` has different behavior/constraints from `fold`.

We'll revisit this when we study empty collections and generic operations more deeply.

---

# 33. Collections and types

Suppose:

```kotlin
val names = listOf("Alice", "Bob", "Charlie")
```

The type is:

```text
List<String>
```

This means:

```text
List
+
elements of type String
```

Similarly:

```kotlin
val numbers = listOf(1, 2, 3)
```

has type roughly:

```text
List<Int>
```

So collections themselves are parameterized by another type.

This is your first important encounter with **generic types**.

We'll study generics properly in a later phase.

For now, recognize:

```text
List<T>
```

means:

> A List containing elements of type `T`.

So:

```text
List<String>
List<Int>
List<Boolean>
```

are all different concrete versions of the general abstraction.

---

# 34. Nested collections

You can even have:

```kotlin
val matrix = listOf(
    listOf(1, 2, 3),
    listOf(4, 5, 6),
    listOf(7, 8, 9)
)
```

Its type is essentially:

```text
List<List<Int>>
```

Now:

```kotlin
matrix[1][2]
```

means:

```text
second list
   ↓
third element
   ↓
6
```

So:

```text
matrix
 ↓
List<List<Int>>
 ↓
list
 ↓
Int
```

This becomes important when dealing with nested data, trees, tables, JSON structures, and other hierarchical models.

---

# 35. Collections can contain nullable elements

For example:

```kotlin
val names: List<String?> = listOf(
    "Alice",
    null,
    "Bob"
)
```

Notice carefully:

```text
List<String?>
```

This means:

> The list itself is non-null, but individual elements may be null.

That's different from:

```text
List<String>?
```

which means:

> The list itself may be null.

These are two different levels of nullability.

```text
List<String?>
→ list exists; elements may be null

List<String>?
→ list may not exist; elements, if present, are Strings
```

This distinction becomes very important in real Kotlin code.

---

# 36. And you can have both

You could theoretically have:

```text
List<String?>?
```

which means:

> the entire list may be null, and elements inside it may also be null.

For example:

```kotlin
val names: List<String?>? = null
```

or:

```kotlin
val names: List<String?>? = listOf("Alice", null, "Bob")
```

At first this can look intimidating.

But just read the type from the inside out:

```text
String?
↓
nullable String

List<String?>
↓
list of nullable Strings

List<String?>?
↓
nullable list of nullable Strings
```

That's a general technique that will help enormously with generic types later.

---

# 37. A real example

Suppose:

```kotlin
val scores = listOf(
    72,
    85,
    91,
    64,
    88
)
```

We want:

> The average of scores above 80.

Write:

```kotlin
val average = scores
    .filter { it > 80 }
    .average()
```

Data flow:

```text
[72, 85, 91, 64, 88]
          ↓
      filter > 80
          ↓
   [85, 91, 88]
          ↓
       average
          ↓
        88.0
```

Now suppose we wanted:

> The names of students whose scores are above 80.

We'd need a relationship between names and scores.

A flat list of integers isn't enough information.

This is where data modeling starts becoming important.

Perhaps:

```text
Alice → 72
Bob → 85
Charlie → 91
David → 64
Eve → 88
```

A `Map` might help.

Or, more realistically, we might eventually create a `Student` class:

```text
Student(name, score)
```

**And that leads directly into the next phase of Kotlin.**

---

# 38. Your exercises

### Exercise 1

Create a `List<Int>` and:

- print its first element,
    
- print its last element,
    
- print its size,
    
- print the element at index `2`.
    

Then deliberately try an invalid index and observe what happens.

Afterward, try `getOrNull()`.

---

### Exercise 2

Create:

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
```

Produce:

```text
[4, 8, 12, 16, 20]
```

using `filter` and `map`.

---

### Exercise 3

Using the same list, determine:

- whether any number is greater than 9,
    
- whether all numbers are positive,
    
- how many are even,
    
- the first number greater than 5.
    

Use the appropriate collection operations rather than manually writing a loop for each question.

---

### Exercise 4

Create:

```kotlin
val ages = mapOf(
    "Alice" to 25,
    "Bob" to 31,
    "Charlie" to 22
)
```

Retrieve:

```text
Alice
Charlie
Unknown
```

Handle the missing key safely.

---

### Exercise 5

Create a mutable set and try inserting the same value multiple times.

Observe the return value of `add()` when:

1. the value is new,
    
2. the value already exists.
    

---

### Exercise 6

Create:

```kotlin
val numbers = listOf(2, 4, 6, 8)
```

Calculate the product using `fold`.

Don't use a manually managed accumulator.

---

# Challenge

Suppose you have:

```kotlin
val prices = listOf(
    120.0,
    450.0,
    80.0,
    700.0,
    250.0,
    50.0
)
```

Calculate:

> The average price of products costing at least `200`.

Do it as a pipeline.

Then calculate:

> The total price of products costing at least `200`.

Then answer this conceptual question:

> Why can `average()` be a problem if the filtered collection is empty?

That question isn't mainly about syntax. It's about **what assumptions an API makes about its input**.

---

# One bigger idea before we move on

At the beginning, collections looked like:

```text
List = bunch of values
Set = bunch of unique values
Map = bunch of key/value pairs
```

Now I want your mental model to be more like:

```text
             Collection
                  │
       ┌──────────┼──────────┐
       ↓          ↓          ↓
      List       Set        Map
       │          │          │
    position   membership   lookup
       │          │          │
       ↓          ↓          ↓
 "what's at     "is it     "what value
 this index?"   present?"   belongs to
                             this key?"
```

And then:

```text
Collection
    ↓
operation
    ↓
new value
```

For example:

```text
List<Int>
   ↓
filter
   ↓
List<Int>
   ↓
map
   ↓
List<Int>
   ↓
sum
   ↓
Int
```

Notice how the **type changes or stays the same depending on the operation**.

That's a very important habit of thought.

---

We've now squeezed a lot out of collections, and there's a natural point to move on.

Until now, most of our data has been primitive things:

```text
Int
String
Boolean
Double
```

But real programs need to represent things like:

```text
User
BankAccount
Product
Order
Student
Car
Payment
```

We need a way to define **our own types**.

That's where we go next:

# **Lesson 17 — Classes and Objects**

We'll start from the problem of representing a real-world entity in code, then build our own types from scratch and understand **properties, constructors, `this`, methods, object identity, and what an object actually means**.