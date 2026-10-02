
# Lesson 11 — Collections: `List`, `Set`, `Map`, and Processing Data

We've reached a point where we can write useful programs, but we're still missing a way to work naturally with **multiple pieces of data**.

Suppose you have 1 student:

```kotlin
val student = "Alice"
```

Fine.

What about 100 students?

You don't want:

```kotlin
val student1 = "Alice"
val student2 = "Bob"
val student3 = "Charlie"
...
```

What we really want is something like:

```text
students
   ↓
[Alice, Bob, Charlie, David, ...]
```

That is the purpose of **collections**.

Kotlin gives us several important collection types, but we'll focus first on three:

```text
List
Set
Map
```

And then we'll learn how Kotlin's higher-order functions let us process them:

```text
map
filter
forEach
find
any
all
fold
...
```

This is a very important area because modern Kotlin code uses collections and collection operations constantly.

---

# 1. What exactly is a collection?

A collection is essentially a structure that holds **multiple values**.

For example:

```kotlin
val numbers = listOf(10, 20, 30, 40)
```

The collection contains four values:

```text
10
20
30
40
```

You can think of it as:

```text
numbers
   ↓
┌────┬────┬────┬────┐
│ 10 │ 20 │ 30 │ 40 │
└────┴────┴────┴────┘
```

But there isn't just one kind of collection.

Different collection types answer different questions.

---

# 2. `List` — ordered values

A `List` represents an ordered sequence of elements.

```kotlin
val numbers = listOf(10, 20, 30, 40)
```

The order matters:

```text
10
20
30
40
```

You can access elements by index:

```kotlin
println(numbers[0])
println(numbers[1])
```

Output:

```text
10
20
```

The first element is at index `0`, not `1`.

So:

```text
index:   0   1   2   3
value:  10  20  30  40
```

This is a common convention in programming.

---

# 3. Why does indexing start at zero?

It's worth understanding rather than just memorizing.

If you think of the first element as having an offset from the beginning:

```text
first element → offset 0
second        → offset 1
third         → offset 2
```

That convention is deeply embedded in many programming languages and data structures.

So:

```kotlin
numbers[0]
```

means:

> Give me the first element.

and:

```kotlin
numbers[3]
```

means:

> Give me the fourth element.

We'll eventually discuss indexing, arrays and performance more deeply.

---

# 4. A `List` can contain duplicate values

For example:

```kotlin
val numbers = listOf(1, 2, 2, 3, 3, 3)
```

All of these are valid elements.

The list preserves:

```text
order
+
duplicates
```

So:

```text
[1, 2, 2, 3, 3, 3]
```

is a perfectly legitimate list.

That distinguishes it from a `Set`.

---

# 5. `Set` — uniqueness matters

Suppose you care only about unique values.

You can use:

```kotlin
val numbers = setOf(1, 2, 3, 2, 1)
```

The resulting set represents the distinct values:

```text
1
2
3
```

A set is useful when the question is:

> Is this value present?

rather than:

> What is the element at position 7?

For example:

```kotlin
val registeredUsers = setOf("Alice", "Bob", "Charlie")

println("Bob" in registeredUsers)
```

produces:

```text
true
```

The key idea is:

```text
List
→ ordered collection, duplicates allowed

Set
→ collection emphasizing uniqueness
```

We'll later discuss the exact ordering and implementation characteristics of different set implementations.

---

# 6. `Map` — key-value relationships

A `Map` is different.

Instead of simply storing:

```text
Alice
Bob
Charlie
```

we may want a relationship like:

```text
Alice   → 25
Bob     → 31
Charlie → 22
```

That's a map.

```kotlin
val ages = mapOf(
    "Alice" to 25,
    "Bob" to 31,
    "Charlie" to 22
)
```

Now we can do:

```kotlin
println(ages["Alice"])
```

which gives:

```text
25
```

Conceptually:

```text
key       value
────────   ─────
Alice  →    25
Bob    →    31
Charlie →   22
```

This is extremely useful for representing associations.

---

# 7. A useful way to choose among them

When you have several pieces of data, ask what relationship you actually need.

### "I care about order."

Use a `List`.

```text
[10, 20, 30, 40]
```

### "I care about uniqueness."

Use a `Set`.

```text
{10, 20, 30, 40}
```

### "I want to look something up using a key."

Use a `Map`.

```text
Alice → 25
Bob → 31
```

This is much more useful than memorizing three APIs independently.

---

# 8. Read-only vs mutable collections

Now we encounter an important Kotlin distinction.

You can write:

```kotlin
val numbers = listOf(1, 2, 3)
```

and:

```kotlin
val numbers = mutableListOf(1, 2, 3)
```

They are not equivalent.

A regular `List` gives you a **read-only interface**.

A `MutableList` gives you operations for changing the collection.

For example:

```kotlin
val numbers = mutableListOf(1, 2, 3)

numbers.add(4)
```

Now:

```text
[1, 2, 3, 4]
```

You can also:

```kotlin
numbers.remove(2)
```

and:

```kotlin
numbers[0] = 100
```

because the collection is mutable.

---

# 9. This is where our `val` discussion becomes important

Consider:

```kotlin
val numbers = mutableListOf(1, 2, 3)
```

We said earlier that `val` means the variable cannot be reassigned.

So:

```kotlin
numbers = mutableListOf(4, 5, 6)
```

is not allowed.

But:

```kotlin
numbers.add(4)
```

is allowed.

Why?

Because these are two different operations:

```text
reassign the variable
```

versus:

```text
modify the object the variable refers to
```

So:

```text
val
↓
reference cannot be reassigned

mutable list
↓
object can still be modified
```

This is exactly the subtle point we previewed earlier.

---

# 10. A read-only collection isn't necessarily "immutable"

This is worth being precise about.

When Kotlin gives you:

```kotlin
List<Int>
```

you're looking at a **read-only interface**.

That doesn't automatically prove that no one anywhere can mutate the underlying collection.

For example, conceptually:

```kotlin
val mutable = mutableListOf(1, 2, 3)
val readOnly: List<Int> = mutable
```

`readOnly` doesn't expose mutation operations.

But `mutable` still can be changed:

```kotlin
mutable.add(4)
```

Then someone reading through `readOnly` can see the changed contents.

So don't equate:

```text
List
```

with:

```text
deep immutable data
```

We'll study immutability and persistent data structures much more carefully later.

For now:

> `List` is read-only from that reference's perspective; `MutableList` exposes mutation operations.

---

# 11. Creating lists

You've already seen:

```kotlin
val numbers = listOf(1, 2, 3)
```

and:

```kotlin
val numbers = mutableListOf(1, 2, 3)
```

You can have an empty list:

```kotlin
val names = emptyList<String>()
```

Or an empty mutable list:

```kotlin
val names = mutableListOf<String>()
```

Notice something interesting:

```kotlin
emptyList<String>()
```

needs the element type because there are no elements from which Kotlin can infer it.

We'll revisit this when we study generics.

---

# 12. Creating sets

```kotlin
val numbers = setOf(1, 2, 3, 2, 1)
```

The conceptual contents are unique:

```text
1
2
3
```

And:

```kotlin
val numbers = mutableSetOf(1, 2, 3)
```

lets you add/remove values.

For example:

```kotlin
numbers.add(4)
numbers.remove(2)
```

---

# 13. Creating maps

```kotlin
val ages = mapOf(
    "Alice" to 25,
    "Bob" to 31,
    "Charlie" to 22
)
```

The `to` here is used to construct key-value pairs for the map.

Then:

```kotlin
println(ages["Bob"])
```

gives:

```text
31
```

You can also use:

```kotlin
ages.containsKey("Alice")
```

to ask whether a key exists.

And:

```kotlin
ages.containsValue(25)
```

to check for a value.

---

# 14. What if the key isn't present?

Consider:

```kotlin
val ages = mapOf(
    "Alice" to 25,
    "Bob" to 31
)
```

Now:

```kotlin
println(ages["Charlie"])
```

doesn't give you some arbitrary integer.

Map lookup through `[]` gives a nullable result because the key might not exist.

Conceptually:

```text
"Bob"     → 31
"Charlie" → nothing
```

We'll revisit this when we combine collections with **null safety**.

That's a nice example of how topics in the course naturally connect.

---

# 15. Iterating over a collection

Now the `for` loop we learned becomes much more useful.

```kotlin
val names = listOf("Alice", "Bob", "Charlie")

for (name in names) {
    println(name)
}
```

Output:

```text
Alice
Bob
Charlie
```

Notice:

We don't care about indexes here.

We're saying:

> For each element in `names`, give it to me as `name`.

This is the more general meaning of `for` that we discussed earlier.

---

# 16. You can iterate over a set

```kotlin
val numbers = setOf(10, 20, 30)

for (number in numbers) {
    println(number)
}
```

Again:

> For each element, execute this code.

The `for` loop doesn't fundamentally care whether the collection is a `List` or `Set`.

It works with things that can be iterated over.

---

# 17. Maps are a little different

A map consists of key-value pairs.

You can write:

```kotlin
val ages = mapOf(
    "Alice" to 25,
    "Bob" to 31
)

for ((name, age) in ages) {
    println("$name is $age")
}
```

Output will represent each key/value pair, such as:

```text
Alice is 25
Bob is 31
```

The syntax:

```kotlin
(name, age)
```

is **destructuring**.

We'll study destructuring more thoroughly later.

For now, think:

```text
map entry
   ↓
key + value
   ↓
name + age
```

---

# 18. Now the interesting part: processing collections

Suppose:

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)
```

and you want:

> Double every number.

You could write:

```kotlin
val result = mutableListOf<Int>()

for (number in numbers) {
    result.add(number * 2)
}
```

This works.

The result becomes:

```text
[2, 4, 6, 8, 10]
```

But Kotlin gives us a much more expressive operation:

```kotlin
val result = numbers.map { it * 2 }
```

And suddenly the code describes the **idea** directly:

> Map each element to twice its value.

This is where the functions-as-values lesson becomes really useful.

---

# 19. `map` transforms elements

Think of:

```kotlin
numbers.map { it * 2 }
```

as:

```text
1 → 2
2 → 4
3 → 6
4 → 8
5 → 10
```

The output is another collection:

```text
[2, 4, 6, 8, 10]
```

So conceptually:

```text
List<A>
   ↓
transformation function
   ↓
List<B>
```

For our example:

```text
List<Int>
   ↓
Int → Int
   ↓
List<Int>
```

This is one of the most important collection concepts to understand.

`map` **doesn't mean "loop."**

More precisely:

> It transforms every element and gives you a collection containing the transformed results.

The fact that iteration happens internally is an implementation detail of how the operation is carried out.

---

# 20. `filter` selects elements

Suppose:

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6)
```

and we want only the even numbers.

We could write a loop:

```kotlin
val result = mutableListOf<Int>()

for (number in numbers) {
    if (number % 2 == 0) {
        result.add(number)
    }
}
```

Or:

```kotlin
val result = numbers.filter { it % 2 == 0 }
```

The lambda:

```kotlin
{ it % 2 == 0 }
```

returns a Boolean.

It's answering:

> Should this element remain in the result?

So:

```text
1 → false
2 → true
3 → false
4 → true
5 → false
6 → true
```

Result:

```text
[2, 4, 6]
```

This is a fundamentally different operation from `map`.

---

# 21. `map` vs `filter`

This distinction is worth remembering deeply.

### `map`

Changes every element.

```text
[1, 2, 3]
   ↓
×2
   ↓
[2, 4, 6]
```

### `filter`

Keeps some elements and discards others.

```text
[1, 2, 3, 4]
   ↓
keep even
   ↓
[2, 4]
```

So:

```text
map
→ one input element produces one output element

filter
→ an input element either survives or doesn't
```

This mental model will serve you better than memorizing method names.

---

# 22. Chaining operations

Here's where Kotlin starts becoming really expressive.

Suppose:

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6)
```

We want:

1. Keep only even numbers.
    
2. Double them.
    

We can write:

```kotlin
val result = numbers
    .filter { it % 2 == 0 }
    .map { it * 2 }
```

Let's follow the data:

```text
[1, 2, 3, 4, 5, 6]
          ↓
       filter
          ↓
[2, 4, 6]
          ↓
        map
          ↓
[4, 8, 12]
```

This is the beginning of a **pipeline**.

A pipeline is simply a sequence of transformations:

```text
input
 ↓
operation
 ↓
operation
 ↓
operation
 ↓
result
```

This style is extremely common in Kotlin.

---

# 23. `forEach`

Sometimes you don't want to create a new collection.

You simply want to perform an action for each item.

For example:

```kotlin
val names = listOf("Alice", "Bob", "Charlie")

names.forEach {
    println(it)
}
```

This prints each name.

Conceptually:

```text
forEach
→ perform an action for each element
```

Compare:

```kotlin
names.map { it.length }
```

which produces a new collection.

So:

```text
map
→ transform data

forEach
→ perform an action
```

That distinction becomes important in real code.

---

# 24. `find`

Suppose:

```kotlin
val numbers = listOf(3, 8, 12, 5, 20)
```

and you want the first number greater than 10.

You can write:

```kotlin
val result = numbers.find { it > 10 }
```

The result is:

```text
12
```

Conceptually:

```text
3  → no
8  → no
12 → yes → return it
```

Notice that `find` doesn't transform every element.

It **searches** for a matching element.

We'll later compare:

```text
find
first
firstOrNull
single
singleOrNull
```

because they have subtly different contracts.

---

# 25. `any`

What if you only care whether **at least one** element satisfies a condition?

```kotlin
val numbers = listOf(3, 8, 12, 5, 20)

val hasLargeNumber = numbers.any { it > 100 }
```

Result:

```text
false
```

Whereas:

```kotlin
val hasLargeNumber = numbers.any { it > 10 }
```

gives:

```text
true
```

The lambda is:

```text
element → Boolean
```

and `any` asks:

> Does at least one element satisfy this condition?

---

# 26. `all`

`all` asks:

> Do all elements satisfy this condition?

For example:

```kotlin
val numbers = listOf(2, 4, 6, 8)

val allEven = numbers.all { it % 2 == 0 }
```

Result:

```text
true
```

But:

```kotlin
val numbers = listOf(2, 4, 5, 8)

val allEven = numbers.all { it % 2 == 0 }
```

gives:

```text
false
```

So now we have three useful patterns:

```text
find
→ give me a matching element

any
→ does at least one match?

all
→ do all match?
```

---

# 27. `none`

The opposite idea:

```kotlin
val numbers = listOf(1, 3, 5, 7)

val noneEven = numbers.none { it % 2 == 0 }
```

Result:

```text
true
```

Meaning:

> No element satisfies the condition.

These operations are expressive because the code describes the question we're asking.

---

# 28. `count`

Suppose:

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6)
```

How many are even?

```kotlin
val count = numbers.count { it % 2 == 0 }
```

Result:

```text
3
```

Conceptually:

```text
1 → no
2 → yes
3 → no
4 → yes
5 → no
6 → yes

count = 3
```

---

# 29. `sum`

For numeric collections:

```kotlin
val numbers = listOf(10, 20, 30)

println(numbers.sum())
```

gives:

```text
60
```

You can combine it with other operations:

```kotlin
val evenSum = numbers
    .filter { it % 2 == 0 }
    .sum()
```

Notice how the pipeline reads naturally:

```text
numbers
→ filter to even numbers
→ sum them
```

That's one of Kotlin's strengths.

---

# 30. Now something deeper: `fold`

You've already used the accumulator pattern with loops.

Remember:

```kotlin
var sum = 0

for (number in numbers) {
    sum += number
}
```

Kotlin gives us a general abstraction for this idea:

```kotlin
val sum = numbers.fold(0) { accumulator, number ->
    accumulator + number
}
```

Let's understand it.

We begin with:

```text
accumulator = 0
```

Then:

```text
0 + 1 = 1
```

Next:

```text
1 + 2 = 3
```

Next:

```text
3 + 3 = 6
```

and so on.

So `fold` essentially means:

> Start with an initial value and repeatedly combine it with each element.

This is much more general than summation.

For example:

```kotlin
val product = numbers.fold(1) { accumulator, number ->
    accumulator * number
}
```

Now we're accumulating multiplication rather than addition.

We'll study `fold`, `reduce`, and the underlying abstraction much more deeply later.

---

# 31. A very useful mental model for collection operations

Rather than memorizing dozens of functions individually, classify them.

### Transform

```text
map
mapNotNull
flatMap
```

Meaning:

> Change the elements.

### Select

```text
filter
filterNot
```

Meaning:

> Decide which elements survive.

### Search

```text
find
first
firstOrNull
```

Meaning:

> Locate an element.

### Ask questions

```text
any
all
none
contains
```

Meaning:

> Determine whether something is true.

### Aggregate

```text
sum
count
fold
reduce
average
```

Meaning:

> Collapse multiple elements into some result.

That conceptual classification is far more valuable than memorizing the names randomly.

---

# 32. A real example

Suppose we have:

```kotlin
val prices = listOf(100.0, 250.0, 80.0, 500.0, 1200.0)
```

We want:

> Find the total price of items costing at least 200.

We can write:

```kotlin
val total = prices
    .filter { it >= 200.0 }
    .sum()
```

Let's follow the data:

```text
[100, 250, 80, 500, 1200]
          ↓
       filter
          ↓
[250, 500, 1200]
          ↓
         sum
          ↓
        1950
```

This is much easier to read than manually managing a loop and accumulator.

---

# 33. Another example: names

```kotlin
val names = listOf(
    "Alice",
    "Bob",
    "Alexander",
    "Charlie",
    "Amanda"
)
```

Suppose we want:

> Names beginning with "A", converted to uppercase.

We can write:

```kotlin
val result = names
    .filter { it.startsWith("A") }
    .map { it.uppercase() }
```

Conceptually:

```text
[Alice, Bob, Alexander, Charlie, Amanda]
                  ↓
               filter
                  ↓
[Alice, Alexander, Amanda]
                  ↓
                 map
                  ↓
[ALICE, ALEXANDER, AMANDA]
```

This is a clean example of composing operations.

---

# 34. An important performance intuition

At this stage, don't obsess over performance.

But notice that:

```kotlin
numbers
    .filter { ... }
    .map { ... }
```

creates intermediate collections with ordinary eager collection operations.

For most everyday tasks, that's completely fine.

Later, we'll learn **sequences**:

```kotlin
numbers
    .asSequence()
    .filter { ... }
    .map { ... }
```

which can process elements lazily and avoid some intermediate allocations.

That's a more advanced topic.

For now, learn normal collections properly first.

---

# 35. Collection operations are not magic

Take:

```kotlin
val result = numbers.map { it * 2 }
```

Don't think:

> "`map` is some special thing Kotlin does."

Think:

> "`map` is a function. I'm passing it another function describing how to transform each element."

Conceptually, something like:

```text
numbers
   ↓
map
   +
(Int) → Int
   ↓
new collection
```

That's exactly the connection between our previous lesson and this one.

We learned:

> Functions can be values.

Now we're using that fact to process data.

---

# 36. Your first genuinely Kotlin-style pipeline

Read this:

```kotlin
val numbers = listOf(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)

val result = numbers
    .filter { it % 2 == 0 }
    .map { it * it }
    .sum()
```

Don't read it as "three functions chained together."

Read it as a sentence:

> Take the numbers, keep the even ones, square them, then add them together.

Let's calculate:

```text
Even numbers:
2, 4, 6, 8, 10

Squares:
4, 16, 36, 64, 100

Sum:
220
```

So:

```text
result = 220
```

This is a style of programming you're going to see constantly in Kotlin.

---

# 37. But loops haven't become useless

Don't fall into the trap of thinking:

> `map` and `filter` are modern, so loops are bad.

No.

Loops are still valuable.

Some operations are clearer as a loop.

Sometimes you need complicated state.

Sometimes you're implementing something that doesn't fit an existing collection operation.

Sometimes a loop is simply easier to understand.

For example:

```kotlin
for (number in numbers) {
    println(number)
}
```

is perfectly readable.

The goal is not:

> Replace every loop with `map`.

The goal is:

> Use the abstraction that best expresses the computation.

---

# 38. `map` has an important promise

A useful mental rule:

For ordinary `map`, each input element produces exactly **one corresponding output element**.

So:

```text
input size = 5
       ↓
map
       ↓
output size = 5
```

For:

```kotlin
listOf(1, 2, 3).map { it * 10 }
```

we get:

```text
[10, 20, 30]
```

Still three elements.

Compare that with `filter`, which can shrink the collection:

```text
input size = 5
       ↓
filter
       ↓
output size could be 0...5
```

That distinction becomes very useful when thinking about pipelines.

---

# 39. `flatMap` — just a preview

Suppose each input produces **multiple** outputs.

For example:

```text
user
  ↓
list of orders
```

One user might have:

```text
[order1, order2, order3]
```

Another:

```text
[order4]
```

If we map users to their order lists, we get a nested structure:

```text
[
  [order1, order2, order3],
  [order4]
]
```

Sometimes we want:

```text
[order1, order2, order3, order4]
```

That's the kind of problem `flatMap` addresses.

We're not going to go deeply into it yet.

I just want you to see that collection transformations can change structure, not just values.

---

# 40. Your exercises

Now I want you to actually write code.

### Exercise 1

Given:

```kotlin
val numbers = listOf(1, 2, 3, 4, 5)
```

create:

```text
[2, 4, 6, 8, 10]
```

using `map`.

---

### Exercise 2

Using the same list, produce:

```text
[2, 4]
```

using `filter`.

---

### Exercise 3

Given:

```kotlin
val numbers = listOf(4, 7, 10, 13, 16, 19)
```

find the first number greater than 15.

---

### Exercise 4

Determine whether:

> at least one number is divisible by 7.

Use `any`.

---

### Exercise 5

Determine whether:

> all numbers are positive.

Use `all`.

---

### Exercise 6

Count how many numbers are even.

Use `count`.

---

### Exercise 7

Calculate the sum of all numbers greater than 10.

Use a pipeline.

---

# Challenge

Given:

```kotlin
val numbers = listOf(
    5, 12, 8, 21, 16, 3, 40, 7, 18
)
```

produce a result containing:

1. only numbers greater than 10,
    
2. only the even ones among them,
    
3. each remaining number squared,
    
4. the final sum.
    

Do it first as a series of intermediate variables so you can see the data changing.

Then rewrite it as one pipeline.

The final answer should be:

```text
?
```

Work it out yourself.

---

# One deeper idea to absorb

We've now connected three major ideas:

```text
Collections
     +
Functions as values
     +
Higher-order functions
```

which gives us:

```text
Data
 ↓
collection
 ↓
operation + behavior
 ↓
transformed data
```

For example:

```kotlin
numbers
    .filter { it > 10 }
    .map { it * 2 }
```

The really important thing isn't that Kotlin has functions called `filter` and `map`.

It's that we're learning to think in terms of:

> **"What transformation do I want to apply to this data?"**

rather than always thinking:

> "How do I manually control every iteration?"

Both styles are useful.

But this abstraction becomes especially powerful once collections get larger and programs become more complex.

---

There's another major question we haven't really addressed yet:

> **What happens when there is no value?**

For example:

```kotlin
val user = findUser("Alice")
```

What if Alice doesn't exist?

Should the function return:

```text
null
```

And if it can return null, how do we safely work with it?

That leads us into one of Kotlin's most important features:

# **Null safety**

We'll finally explore `null`, nullable types like `String?`, safe calls `?.`, the Elvis operator `?:`, smart casts, `let`, and why Kotlin tries to move an entire category of bugs from runtime into compile time.