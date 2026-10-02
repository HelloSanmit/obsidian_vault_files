
# Lesson 18 — `data class`: When an Object Represents Data

In the previous lesson, we created an ordinary class:

```kotlin
class Person(
    val name: String,
    val age: Int
)
```

Then we noticed something interesting.

Suppose we create:

```kotlin
val person1 = Person("Alice", 25)
val person2 = Person("Alice", 25)
```

They contain exactly the same information.

But they're still two separately created objects.

That raises a fundamental question:

> When we're representing something like a user, product, student, address, or API response, do we really care about the **identity of the object**, or do we primarily care about the **data it contains**?

For many such types, we care about the data.

That's exactly the problem `data class` is designed to solve.

---

# 1. Your first `data class`

Instead of:

```kotlin
class Person(
    val name: String,
    val age: Int
)
```

we can write:

```kotlin
data class Person(
    val name: String,
    val age: Int
)
```

That one word:

```text
data
```

changes some important behavior.

Kotlin automatically provides useful functionality for data classes, including `equals()`, `hashCode()`, `toString()`, `componentN()` functions, and `copy()`. ([Kotlin](https://kotlinlang.org/docs/data-classes.html?utm_source=chatgpt.com "Data classes | Kotlin Documentation"))

And this is exactly why data classes are so common in real Kotlin code.

---

# 2. The most important difference: equality

Let's compare ordinary classes first.

```kotlin
class Person(
    val name: String,
    val age: Int
)
```

Then:

```kotlin
val person1 = Person("Alice", 25)
val person2 = Person("Alice", 25)

println(person1 == person2)
```

For a normal class that hasn't overridden equality, `==` ultimately uses the class's `equals()` behavior, which by default is inherited from `Any` and behaves as referential equality. So these two separately created objects aren't equal merely because their fields match. ([Kotlin](https://kotlinlang.org/docs/equality.html?utm_source=chatgpt.com "Equality | Kotlin Documentation"))

Now change the class to:

```kotlin
data class Person(
    val name: String,
    val age: Int
)
```

and:

```kotlin
val person1 = Person("Alice", 25)
val person2 = Person("Alice", 25)

println(person1 == person2)
```

Now the answer is:

```text
true
```

Why?

Because data classes automatically generate structural equality based on the properties in the **primary constructor**. ([Kotlin](https://kotlinlang.org/docs/data-classes.html?utm_source=chatgpt.com "Data classes | Kotlin Documentation"))

So:

```text
Person("Alice", 25)
```

is considered equal to another:

```text
Person("Alice", 25)
```

because their data matches.

---

# 3. This connects directly to `==` vs `===`

We've talked about this before, but now it finally becomes concrete.

Consider:

```kotlin
data class Person(
    val name: String,
    val age: Int
)

val person1 = Person("Alice", 25)
val person2 = Person("Alice", 25)

println(person1 == person2)
println(person1 === person2)
```

You'll get:

```text
true
false
```

Because:

```text
== 
↓
structural equality
↓
"Do these objects have equal contents according to equals()?"
```

whereas:

```text
===
↓
referential equality
↓
"Are these references pointing to the exact same object?"
```

Kotlin explicitly distinguishes these two forms of equality. ([Kotlin](https://kotlinlang.org/docs/equality.html?utm_source=chatgpt.com "Equality | Kotlin Documentation"))

So:

```text
same data
≠
same object
```

A data class is saying:

> For this type, the data is important enough that equality should normally be based on it.

---

# 4. Think about a `User`

Suppose you're building an application.

You receive:

```text
id = 42
name = "Alice"
email = "alice@example.com"
```

You might represent it as:

```kotlin
data class User(
    val id: Int,
    val name: String,
    val email: String
)
```

Now imagine data comes from a database:

```kotlin
val userFromDatabase = User(
    42,
    "Alice",
    "alice@example.com"
)
```

and later an HTTP API gives you:

```kotlin
val userFromApi = User(
    42,
    "Alice",
    "alice@example.com"
)
```

They may be two different object instances, but from the application's perspective they represent the same data.

A `data class` makes that kind of comparison natural.

This is why data classes are particularly common for:

- API models
    
- database records
    
- configuration data
    
- UI state
    
- request/response objects
    
- domain values
    

---

# 5. `toString()` comes for free

Try this with an ordinary class:

```kotlin
class Person(
    val name: String,
    val age: Int
)

val person = Person("Alice", 25)

println(person)
```

You may get an implementation-oriented representation rather than something useful for reading.

Now with:

```kotlin
data class Person(
    val name: String,
    val age: Int
)
```

and:

```kotlin
println(person)
```

you get something like:

```text
Person(name=Alice, age=25)
```

Kotlin generates a readable `toString()` representation using the data properties. ([Kotlin](https://kotlinlang.org/docs/data-classes.html?utm_source=chatgpt.com "Data classes | Kotlin Documentation"))

This is incredibly useful when debugging.

Imagine logging:

```kotlin
println(user)
```

and immediately seeing:

```text
User(id=42, name=Alice, email=alice@example.com)
```

instead of some opaque object identifier.

---

# 6. `data class` isn't magic

It's tempting to think:

> "The compiler somehow knows everything important about this class."

It doesn't.

There are specific rules about **which properties count as data properties**.

The most important rule for us right now is:

> The properties declared in the primary constructor participate in the generated `equals()`, `hashCode()`, `toString()`, `copy()`, and `componentN()` behavior. ([Kotlin](https://kotlinlang.org/docs/data-classes.html?utm_source=chatgpt.com "Data classes | Kotlin Documentation"))

So this matters:

```kotlin
data class Person(
    val name: String,
    val age: Int
)
```

Both `name` and `age` are data properties.

---

# 7. Properties declared in the class body behave differently

Consider:

```kotlin
data class Person(
    val name: String
) {
    var age: Int = 0
}
```

Now:

```kotlin
val person1 = Person("Alice")
val person2 = Person("Alice")

person1.age = 20
person2.age = 30

println(person1 == person2)
```

This prints:

```text
true
```

Why?

Because `age` isn't part of the primary constructor.

Therefore it isn't included in the automatically generated equality/hashCode/toString/component behavior. ([Kotlin](https://kotlinlang.org/docs/data-classes.html?utm_source=chatgpt.com "Data classes | Kotlin Documentation"))

So:

```text
data class Person(val name: String) {
    var age: Int = 0
}
```

essentially tells Kotlin:

> `name` defines the data identity for the generated data-class operations.

That can be useful, but it can also be a design trap if you accidentally put important identity-related state outside the constructor.

---

# 8. `hashCode()`

You've probably encountered `equals()`.

You may have seen:

```kotlin
hashCode()
```

and wondered what it has to do with equality.

The short conceptual version is:

> `hashCode()` produces an integer hash associated with an object according to its equality semantics.

The important contract is:

> If two objects are equal according to `equals()`, they must have the same hash code.

Kotlin data classes automatically generate a matching `equals()` / `hashCode()` pair. ([Kotlin](https://kotlinlang.org/docs/data-classes.html?utm_source=chatgpt.com "Data classes | Kotlin Documentation"))

Why does this matter?

Because hash-based collections such as sets and maps rely on hashing.

For example:

```kotlin
val users = setOf(
    User(1, "Alice"),
    User(1, "Alice")
)
```

A correctly implemented data class can recognize those as equal values rather than unrelated objects.

We'll explore hash collections in much greater depth later.

For now:

```text
equals()
↓
"Are these values equal?"

hashCode()
↓
"Which hash bucket should this value participate in?"
```

That's a simplification, but a useful one.

---

# 9. `copy()`

Here's another extremely useful feature.

Suppose:

```kotlin
data class User(
    val name: String,
    val age: Int
)
```

and:

```kotlin
val user = User("Alice", 25)
```

You want a new user with the same information except the age:

```kotlin
val olderUser = user.copy(age = 26)
```

Now:

```text
user
→ Alice, 25

olderUser
→ Alice, 26
```

Kotlin generates the `copy()` function automatically for data classes. ([Kotlin](https://kotlinlang.org/docs/data-classes.html?utm_source=chatgpt.com "Data classes | Kotlin Documentation"))

This is particularly valuable when working with immutable data.

---

# 10. Why is `copy()` useful?

Suppose you have:

```kotlin
val state = UserState(
    name = "Alice",
    isLoggedIn = false,
    notificationCount = 3
)
```

You want to represent a new state where the user is logged in:

```kotlin
val newState = state.copy(
    isLoggedIn = true
)
```

You don't have to rewrite everything:

```kotlin
UserState(
    name = state.name,
    isLoggedIn = true,
    notificationCount = state.notificationCount
)
```

`copy()` expresses the intention:

> Create another version of this data, changing only what I specify.

You'll see this pattern constantly in modern Kotlin architecture, especially when modeling immutable UI state.

---

# 11. `copy()` is shallow

This is an important subtlety.

Consider:

```kotlin
data class User(
    val name: String,
    val hobbies: MutableList<String>
)
```

Now:

```kotlin
val user1 = User(
    "Alice",
    mutableListOf("Reading")
)

val user2 = user1.copy()
```

It may be tempting to think:

> "user2 has its own independent copy of the hobbies list."

Not necessarily.

The generated `copy()` is **shallow**. References held by the copied object are shared unless you explicitly create new nested objects yourself. ([Kotlin](https://kotlinlang.org/docs/data-classes.html?utm_source=chatgpt.com "Data classes | Kotlin Documentation"))

So conceptually:

```text
user1 ──→ hobbies list ←── user2
```

Both can refer to the same mutable list.

Therefore:

```kotlin
user2.hobbies.add("Gaming")
```

can also affect what you observe through `user1.hobbies`.

This is a very important distinction:

```text
copy object
≠
deep copy everything inside object
```

We'll revisit this when we study references, mutability, and immutable data structures.

---

# 12. Destructuring

Data classes also give us another nice feature.

Suppose:

```kotlin
data class User(
    val name: String,
    val age: Int
)
```

and:

```kotlin
val user = User("Alice", 25)
```

You can write:

```kotlin
val (name, age) = user
```

Now:

```text
name → "Alice"
age  → 25
```

This is called **destructuring**.

Kotlin's data classes automatically provide `componentN()` functions that make this possible. ([Kotlin](https://kotlinlang.org/docs/data-classes.html?utm_source=chatgpt.com "Data classes | Kotlin Documentation"))

---

# 13. What does `component1()` mean?

Underneath the destructuring syntax:

```kotlin
val (name, age) = user
```

Kotlin conceptually uses:

```kotlin
val name = user.component1()
val age = user.component2()
```

The generated component functions correspond to the constructor properties in their declared order. ([Kotlin](https://kotlinlang.org/docs/destructuring-declarations.html?utm_source=chatgpt.com "Destructuring declarations | Kotlin Documentation"))

So:

```text
component1()
→ name

component2()
→ age
```

This is an example of a broader Kotlin idea called a **convention**.

You've already encountered conventions through operator syntax.

Kotlin often provides convenient syntax that maps to specially named functions behind the scenes.

We'll study Kotlin's convention system later.

---

# 14. Order matters in destructuring

Suppose:

```kotlin
data class User(
    val name: String,
    val age: Int
)
```

Then:

```kotlin
val (name, age) = user
```

works as expected.

But:

```kotlin
val (age, name) = user
```

does **not** somehow inspect the variable names and figure out what you intended.

The destructuring positions correspond to:

```text
component1 → first property
component2 → second property
```

So you'd actually get:

```text
age  ← "Alice"
name ← 25
```

which is obviously wrong from a domain perspective.

This is why destructuring is positional in its normal form. ([Kotlin](https://kotlinlang.org/docs/destructuring-declarations.html?utm_source=chatgpt.com "Destructuring declarations | Kotlin Documentation"))

---

# 15. You can ignore a component

Suppose:

```kotlin
val user = User("Alice", 25)
```

but you only care about the name.

You can write:

```kotlin
val (name, _) = user
```

The underscore means:

> I don't need this component.

This is especially useful in collections and map iteration.

For example:

```kotlin
val scores = mapOf(
    "Alice" to 90,
    "Bob" to 80
)

for ((name, _) in scores) {
    println(name)
}
```

Again, this is just another bit of expressive Kotlin syntax built around the underlying component convention. ([Kotlin](https://kotlinlang.org/docs/destructuring-declarations.html?utm_source=chatgpt.com "Destructuring declarations | Kotlin Documentation"))

---

# 16. Data classes work nicely with maps

Remember:

```kotlin
val ages = mapOf(
    "Alice" to 25,
    "Bob" to 30
)
```

Because map entries expose appropriate component functions, you can write:

```kotlin
for ((name, age) in ages) {
    println("$name is $age")
}
```

Conceptually, each entry gives you:

```text
component1 → key
component2 → value
```

This is why the syntax feels so natural.

---

# 17. When should you use a data class?

A good first intuition is:

> Use a `data class` when the primary purpose of the type is to represent **data/value state**.

Examples:

```kotlin
data class User(
    val id: Int,
    val name: String
)
```

```kotlin
data class Product(
    val id: Int,
    val name: String,
    val price: Double
)
```

```kotlin
data class Coordinates(
    val latitude: Double,
    val longitude: Double
)
```

```kotlin
data class OrderItem(
    val productId: Int,
    val quantity: Int
)
```

The emphasis is on the information represented.

---

# 18. Not every class should be a data class

Suppose we have:

```kotlin
class BankAccount(
    val accountNumber: String,
    private var balance: Double
)
```

The identity and behavior of a bank account may be more important than simply saying:

> Two account objects are equal whenever all their current fields match.

The account has:

- identity,
    
- mutable state,
    
- operations,
    
- invariants,
    
- lifecycle.
    

That may make a normal class a better conceptual model.

Compare:

```text
User profile data
→ data class makes natural sense

Bank account entity
→ identity + behavior may matter more
```

There isn't a universal rule, but don't put `data` in front of every class automatically.

---

# 19. Value vs entity

We're touching an important software-design concept.

Some objects behave more like **values**:

```text
Money(100, INR)
Coordinates(18.52, 73.85)
Date(2026, 9, 18)
Address(...)
```

If two separately created objects contain the same meaningful data, we may want to consider them equal.

Other objects behave more like **entities**:

```text
BankAccount
UserAccount
Order
Employee
```

where identity can matter independently from current field values.

This distinction is not absolute, but it's a useful design lens:

```text
value-like object
→ equality based on data

entity-like object
→ identity may matter
```

Data classes are particularly suited to the first category.

---

# 20. Data classes and immutability work nicely together

Consider:

```kotlin
data class User(
    val name: String,
    val age: Int
)
```

Because the properties are `val`, the data itself can't be reassigned after creation.

You can create a modified version with:

```kotlin
val updatedUser = user.copy(age = 26)
```

So instead of:

```text
same object
↓
mutate state repeatedly
```

you can often model:

```text
old value
   ↓
copy with modification
   ↓
new value
```

That style is especially useful in functional programming and modern application state management.

We'll see this idea repeatedly when we study coroutines, Flow and Android state.

---

# 21. `data class` doesn't make its properties immutable

Just like `val` doesn't automatically make an object immutable, `data class` doesn't magically make everything inside it immutable.

For example:

```kotlin
data class Team(
    val name: String,
    val members: MutableList<String>
)
```

The `members` property cannot be reassigned:

```kotlin
team.members = ...
```

but the list itself can still change:

```kotlin
team.members.add("Charlie")
```

So:

```text
data class
≠
immutable object
```

and:

```text
val
≠
deep immutability
```

These are separate concepts.

That's an important lesson to carry forward.

---

# 22. Data classes have some language restrictions

Kotlin requires a data class to have at least one parameter in its primary constructor, and every primary-constructor parameter must be marked `val` or `var`. Data classes also cannot be `abstract`, `open`, `sealed`, or `inner`. ([Kotlin](https://kotlinlang.org/docs/data-classes.html?utm_source=chatgpt.com "Data classes | Kotlin Documentation"))

For example:

```kotlin
data class Person()
```

isn't a valid data class because it has no primary-constructor parameter.

And:

```kotlin
data class Person(
    name: String
)
```

isn't valid as a data class because `name` isn't declared as `val` or `var`.

But:

```kotlin
data class Person(
    val name: String
)
```

is valid.

---

# 23. A data class can still have functions

Don't think:

> "Data class means it can't have behavior."

It absolutely can.

For example:

```kotlin
data class Rectangle(
    val width: Double,
    val height: Double
) {
    val area: Double
        get() = width * height

    fun scale(factor: Double): Rectangle {
        return copy(
            width = width * factor,
            height = height * factor
        )
    }
}
```

You can have methods and computed properties.

The word `data` primarily affects the automatically generated data-oriented operations.

It doesn't prohibit normal class functionality. ([Kotlin](https://kotlinlang.org/docs/data-classes.html?utm_source=chatgpt.com "Data classes | Kotlin Documentation"))

---

# 24. Let's build a useful example

Imagine an e-commerce product:

```kotlin
data class Product(
    val id: Int,
    val name: String,
    val price: Double
)
```

Now:

```kotlin
val product = Product(
    id = 101,
    name = "Keyboard",
    price = 2500.0
)
```

Printing:

```kotlin
println(product)
```

gives readable information.

Copying:

```kotlin
val discounted = product.copy(
    price = 2000.0
)
```

Equality:

```kotlin
println(product == discounted)
```

gives:

```text
false
```

because the data differs.

And:

```kotlin
val sameProduct = Product(
    id = 101,
    name = "Keyboard",
    price = 2500.0
)

println(product == sameProduct)
```

gives:

```text
true
```

even though `product` and `sameProduct` are different object instances.

That is exactly the behavior we wanted.

---

# 25. Let's compare the two models

### Ordinary class

```kotlin
class Product(
    val id: Int,
    val name: String,
    val price: Double
)
```

Two separately created instances don't automatically become equal merely because their fields match.

### Data class

```kotlin
data class Product(
    val id: Int,
    val name: String,
    val price: Double
)
```

Their equality is derived from the primary-constructor properties.

So:

```text
ordinary class
→ identity-oriented by default

data class
→ data/value-oriented by default
```

That's the conceptual distinction I want you to remember.

---

# 26. Don't memorize "data class = getters/setters"

This is a common misunderstanding.

A data class isn't special because it automatically gives you some trivial accessors.

Its real importance is its **value semantics**.

The generated pieces work together:

```text
equals()
hashCode()
toString()
copy()
componentN()
```

to make the object convenient to use as a representation of data. ([Kotlin](https://kotlinlang.org/docs/data-classes.html?utm_source=chatgpt.com "Data classes | Kotlin Documentation"))

That combination is the important idea.

---

# 27. A deeper connection to collections

Suppose:

```kotlin
data class User(
    val id: Int,
    val name: String
)
```

Then:

```kotlin
val users = listOf(
    User(1, "Alice"),
    User(2, "Bob"),
    User(3, "Charlie")
)
```

Now collection operations become natural:

```kotlin
val alice = users.find { it.name == "Alice" }
```

And equality becomes useful:

```kotlin
val expected = User(1, "Alice")
val actual = users.first()

println(actual == expected)
```

This works structurally because the data class defines equality based on its data properties.

This is one reason data classes are so useful in testing.

---

# 28. A test becomes much easier to express

Suppose a function produces:

```kotlin
User(1, "Alice")
```

Instead of testing individual fields:

```kotlin
assertEquals(1, result.id)
assertEquals("Alice", result.name)
```

you can often compare the entire data value against the expected one:

```kotlin
assertEquals(
    User(1, "Alice"),
    result
)
```

The structural equality of data classes makes that natural.

We'll make much heavier use of this when we reach testing.

---

# 29. One more subtle point: property order matters

Because `componentN()` follows constructor-property order:

```kotlin
data class User(
    val name: String,
    val age: Int
)
```

has:

```text
component1() → name
component2() → age
```

If you change the constructor to:

```kotlin
data class User(
    val age: Int,
    val name: String
)
```

the components change accordingly.

This is another reason not to use destructuring everywhere merely because it's concise.

Sometimes:

```kotlin
val (name, age) = user
```

is excellent.

But:

```kotlin
val (x, y, z, a, b, c) = hugeObject
```

can become very difficult to read.

As always:

> Concision is useful only when it preserves understanding.

---

# 30. Your exercises

### Exercise 1 — Equality

Create:

```kotlin
data class Person(
    val name: String,
    val age: Int
)
```

Create:

```kotlin
val p1 = Person("Alice", 25)
val p2 = Person("Alice", 25)
val p3 = Person("Bob", 25)
```

Predict:

```kotlin
p1 == p2
p1 == p3
p1 === p2
```

Then run it.

Explain each result in your own words.

---

### Exercise 2 — `copy`

Create:

```kotlin
val person = Person("Alice", 25)
```

Then create a second person who is 26 using `copy()`.

Verify that the original remains unchanged.

---

### Exercise 3 — destructuring

Write:

```kotlin
val person = Person("Alice", 25)
```

and destructure it:

```kotlin
val (name, age) = person
```

Print both.

Then reverse the variable names and see what actually happens.

---

### Exercise 4 — body property

Create:

```kotlin
data class Person(
    val name: String
) {
    var age: Int = 0
}
```

Create two people with the same name but different ages.

Check:

```kotlin
person1 == person2
```

and explain why the result is what it is.

---

### Exercise 5 — shallow copy

Create:

```kotlin
data class User(
    val name: String,
    val hobbies: MutableList<String>
)
```

Make a copy.

Add an item through the copied object's list.

Then inspect both objects.

Try to explain what happened **without using the word "because Kotlin copied it."**

Instead, think in terms of references.

That distinction will prepare you for a much deeper lesson later.

---

# Challenge

Design a:

```kotlin
data class ShoppingItem(
    ...
)
```

with sensible properties.

Then create a list of several items.

Use collection operations to:

1. find an item,
    
2. filter expensive items,
    
3. calculate the total price,
    
4. create a modified copy of one item,
    
5. compare the original and modified item.
    

This combines almost the entire course so far:

```text
classes
+
types
+
collections
+
lambdas
+
functions
+
equality
+
copying
```

---

# The deeper idea

We started with:

```text
class
↓
object
```

and then discovered that objects can have different kinds of meaning.

Sometimes an object represents an **entity**:

> "This particular thing has its own identity."

Sometimes an object represents a **value**:

> "I care about what information this thing contains."

A `data class` is Kotlin's convenient way of saying:

> **This type is primarily about its data.**

That's why Kotlin generates:

```text
equals()
hashCode()
toString()
copy()
componentN()
```

for you. ([Kotlin](https://kotlinlang.org/docs/data-classes.html?utm_source=chatgpt.com "Data classes | Kotlin Documentation"))

And this is the first time we're really seeing Kotlin's language features helping us **model meaning**, not just write shorter code.

---

There's one more important piece missing from our object model.

We've created classes, but our classes are still mostly isolated types:

```text
Person
Product
BankAccount
```

What if several types share common behavior?

For example:

```text
Car
Bike
Truck
```

might all be able to:

```text
start()
stop()
```

And what if different classes should be usable through a common type?

That's where we get into one of the central ideas of object-oriented programming:

# **Interfaces, inheritance, and polymorphism**

We'll start by asking **why inheritance exists at all**, then learn interfaces, `open`, `override`, abstract classes, polymorphism, and composition—and importantly, when inheritance is actually the wrong tool.