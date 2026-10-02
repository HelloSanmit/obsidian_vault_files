
# Lesson 17 — Classes and Objects

Up to now, we've mostly represented information using primitive/basic types:

```kotlin
val name = "Alice"
val age = 25
val isActive = true
```

And we've used collections to group data:

```kotlin
val names = listOf("Alice", "Bob", "Charlie")
```

But real applications deal with **things that have multiple pieces of related information and behavior**.

Think about a bank account.

A bank account might have:

```text
account number
owner
balance
currency
```

and it can do things:

```text
deposit money
withdraw money
check balance
close account
```

It would be awkward to represent that using unrelated variables:

```kotlin
val accountNumber = "12345"
val owner = "Alice"
var balance = 5000.0
val currency = "INR"
```

What happens when you have 10,000 accounts?

You need a way to say:

> "These pieces of data belong together, and these operations belong to that same concept."

That's what **classes and objects** are about.

---

# 1. Your first class

Let's define a `Person`.

```kotlin
class Person {
    var name = ""
    var age = 0
}
```

We've just created a new **type**:

```text
Person
```

Just like Kotlin already gives us:

```text
Int
String
Boolean
```

we have now created our own:

```text
Person
```

This is a huge step.

Before:

```text
Kotlin gives us the type
```

Now:

```text
We define the type ourselves
```

---

# 2. Creating an object

A class is a definition.

To actually create a person, we instantiate the class:

```kotlin
val person = Person()
```

Now:

```text
Person
   ↓
object
   ↓
person
```

You can think of the class as a **blueprint** and the object as an actual instance built from that blueprint.

For example:

```text
Class: Person

        ↓

Object 1: Alice, 25
Object 2: Bob, 31
Object 3: Charlie, 22
```

All are `Person`s, but each object has its own state.

---

# 3. Using the object's properties

With:

```kotlin
class Person {
    var name = ""
    var age = 0
}
```

we can do:

```kotlin
fun main() {
    val person = Person()

    person.name = "Alice"
    person.age = 25

    println(person.name)
    println(person.age)
}
```

Output:

```text
Alice
25
```

The syntax:

```kotlin
person.name
```

means:

> Access the `name` property belonging to this particular `person` object.

Likewise:

```kotlin
person.age
```

accesses that object's `age`.

---

# 4. One class, multiple objects

This is where classes become useful.

```kotlin
class Person {
    var name = ""
    var age = 0
}
```

Now:

```kotlin
val person1 = Person()
val person2 = Person()

person1.name = "Alice"
person1.age = 25

person2.name = "Bob"
person2.age = 30
```

We now have two different objects:

```text
person1
   ↓
Person
name = Alice
age = 25


person2
   ↓
Person
name = Bob
age = 30
```

Changing one does not automatically change the other:

```kotlin
person1.age = 26
```

Now:

```text
person1.age → 26
person2.age → 30
```

They are separate object instances.

---

# 5. Class vs object

This distinction is extremely important.

A **class** describes what a type looks like and what it can do.

An **object** is an actual instance of that class.

So:

```text
class Person
```

is a definition.

While:

```kotlin
val person = Person()
```

creates an object.

Think:

```text
Class
↓
recipe/blueprint

Object
↓
actual thing created from it
```

A class isn't itself "Alice".

It describes what a `Person` is.

The object contains the actual state for one particular person.

---

# 6. Let's improve the class

Our previous version requires this:

```kotlin
val person = Person()

person.name = "Alice"
person.age = 25
```

But it would be nicer if we could construct the person with its information immediately.

That's where a **constructor** comes in.

```kotlin
class Person(
    val name: String,
    val age: Int
)
```

Now we can write:

```kotlin
val person = Person("Alice", 25)
```

Much cleaner.

We're saying:

> To construct a `Person`, provide a name and an age.

---

# 7. Primary constructor

This part:

```kotlin
class Person(
    val name: String,
    val age: Int
)
```

is called the **primary constructor**.

It may look unfamiliar because Kotlin puts constructor parameters directly in the class header.

Compare it conceptually with:

```text
Person needs:
    name : String
    age  : Int
```

And:

```kotlin
Person("Alice", 25)
```

provides those values.

This is one of Kotlin's most convenient language features.

---

# 8. Why is there a `val` in the constructor?

This is important.

Consider:

```kotlin
class Person(
    val name: String,
    val age: Int
)
```

The `val` makes these constructor parameters into **properties of the object**.

So you can do:

```kotlin
val person = Person("Alice", 25)

println(person.name)
println(person.age)
```

But compare:

```kotlin
class Person(
    name: String,
    age: Int
)
```

Now `name` and `age` are constructor parameters, but not automatically properties.

You can't simply do:

```kotlin
person.name
```

because no `name` property was declared.

This distinction is extremely important:

```text
constructor parameter
≠
property
```

unless you explicitly make it one, such as with `val` or `var`.

---

# 9. `val` vs `var` in constructor properties

You can write:

```kotlin
class Person(
    val name: String,
    var age: Int
)
```

Now:

```kotlin
val person = Person("Alice", 25)

person.age = 26
```

is valid.

But:

```kotlin
person.name = "Bob"
```

is not, because `name` is a `val`.

So the same rule we learned earlier applies to properties:

```text
val → cannot reassign property
var → can reassign property
```

---

# 10. A class can contain properties and functions

For example:

```kotlin
class Person(
    val name: String,
    var age: Int
) {
    fun introduce() {
        println("Hi, I'm $name and I'm $age years old.")
    }
}
```

Now:

```kotlin
fun main() {
    val person = Person("Alice", 25)

    person.introduce()
}
```

Output:

```text
Hi, I'm Alice and I'm 25 years old.
```

We've now put **data and behavior together**.

The object has:

```text
state:
name
age

behavior:
introduce()
```

That's one of the central ideas of object-oriented programming.

---

# 11. A method is just a function belonging to a class

We already learned functions:

```kotlin
fun introduce() {
    ...
}
```

When that function is defined inside a class, we commonly call it a **method**.

For example:

```kotlin
class Person(
    val name: String
) {
    fun greet() {
        println("Hello, $name")
    }
}
```

Then:

```kotlin
person.greet()
```

The important conceptual point is:

> A method has access to the state of the object it belongs to.

That's why it can use:

```kotlin
name
```

without requiring:

```kotlin
name: String
```

as a separate parameter.

---

# 12. The mysterious `this`

Inside a class, Kotlin provides:

```kotlin
this
```

which refers to the **current object instance**.

For example:

```kotlin
class Person(
    val name: String,
    var age: Int
) {
    fun introduce() {
        println("Name: ${this.name}")
        println("Age: ${this.age}")
    }
}
```

Here:

```kotlin
this.name
```

means:

> The `name` property of this particular object.

Often you don't need to write `this`:

```kotlin
println(name)
```

is enough.

But `this` becomes particularly useful when names overlap.

---

# 13. Why `this` matters

Consider:

```kotlin
class Person(
    var name: String
) {
    fun rename(name: String) {
        this.name = name
    }
}
```

Here there are two `name`s:

```text
this.name
   ↓
property of the object

name
   ↓
parameter of the function
```

So:

```kotlin
this.name = name
```

means:

> Assign the parameter `name` to the object's `name` property.

Without `this`, the statement would be ambiguous in meaning to a human reader.

This pattern appears frequently in object-oriented code.

---

# 14. An object's state

Let's make this concrete.

```kotlin
class BankAccount(
    val owner: String,
    var balance: Double
)
```

Create:

```kotlin
val account = BankAccount("Alice", 1000.0)
```

The object currently has:

```text
owner   → Alice
balance → 1000.0
```

Now:

```kotlin
account.balance += 500.0
```

The same object now has:

```text
owner   → Alice
balance → 1500.0
```

The object has **state**.

State means:

> The data associated with an object at a particular point in time.

This concept will become very important throughout the course.

---

# 15. Give the object behavior

Instead of changing the balance from outside:

```kotlin
account.balance -= 200.0
```

we could make the object responsible for withdrawing:

```kotlin
class BankAccount(
    val owner: String,
    var balance: Double
) {
    fun deposit(amount: Double) {
        balance += amount
    }

    fun withdraw(amount: Double) {
        balance -= amount
    }
}
```

Now:

```kotlin
account.deposit(500.0)
account.withdraw(200.0)
```

The operations are attached to the concept that owns the state.

This is a major idea in object-oriented design:

> **Put behavior close to the data it naturally operates on.**

We'll later discuss when this is a good idea and when a separate service/function is more appropriate.

---

# 16. Encapsulation

Suppose we don't want code outside the class to freely modify the balance.

We could make it:

```kotlin
class BankAccount(
    val owner: String,
    private var balance: Double
) {
    fun deposit(amount: Double) {
        balance += amount
    }

    fun withdraw(amount: Double) {
        balance -= amount
    }

    fun getBalance(): Double {
        return balance
    }
}
```

Now:

```kotlin
account.balance = -100000.0
```

isn't allowed from outside.

The `balance` property is:

```text
private
```

This is **encapsulation**:

> The class controls access to its internal state.

The outside world interacts through the operations the class chooses to expose.

This becomes extremely important for designing maintainable software.

---

# 17. Why hide internal state?

Suppose a bank account allows:

```text
balance = -500000
```

with no validation.

If every part of your application can directly modify:

```kotlin
account.balance
```

then you have no central place enforcing the rules.

But if modification happens through:

```kotlin
account.withdraw(500.0)
```

the class can enforce:

```kotlin
if (amount > balance) {
    ...
}
```

Now the class protects its own invariants.

An **invariant** is a condition that should remain true for an object.

For example:

> A bank account's balance must never become negative.

Whether that's actually a valid business rule depends on the type of account, but the idea is what's important.

---

# 18. Properties don't have to be publicly mutable

A very common design is:

```kotlin
class User(
    val name: String,
    private var _age: Int
) {
    val age: Int
        get() = _age
}
```

Outside code can read:

```kotlin
user.age
```

but cannot assign:

```kotlin
user.age = 30
```

This is one way to expose information without exposing unrestricted mutation.

There are more elegant ways to model this, including private setters, which we'll cover later.

---

# 19. Classes can contain computed properties

A property doesn't necessarily have to store a separate value.

For example:

```kotlin
class Rectangle(
    val width: Double,
    val height: Double
) {
    val area: Double
        get() = width * height
}
```

Now:

```kotlin
val rectangle = Rectangle(10.0, 5.0)

println(rectangle.area)
```

produces:

```text
50.0
```

Notice:

```kotlin
area
```

looks like a property:

```kotlin
rectangle.area
```

but its value is computed from:

```text
width × height
```

This is called a **custom getter**.

We'll study property accessors much more deeply later.

---

# 20. A constructor can perform initialization logic

Sometimes simply receiving constructor parameters isn't enough.

Suppose:

```kotlin
class Person(
    val name: String,
    val age: Int
)
```

What if `age` must not be negative?

You can write an `init` block:

```kotlin
class Person(
    val name: String,
    val age: Int
) {
    init {
        require(age >= 0)
    }
}
```

Now:

```kotlin
val person = Person("Alice", 25)
```

works.

But:

```kotlin
val person = Person("Alice", -5)
```

fails.

The `init` block runs when the object is initialized.

This is useful for enforcing invariants when an object is created.

---

# 21. Why validation at construction matters

Imagine your class allows:

```kotlin
class Person(
    val name: String,
    val age: Int
)
```

and the rest of your program assumes:

```text
age >= 0
```

If you don't enforce that assumption, you could accidentally create:

```text
Person("Alice", -200)
```

Then every function using `Person` has to wonder:

> Could age be invalid?

But if construction guarantees:

```text
age >= 0
```

then the rest of the program can reason with a stronger assumption.

This is an important software design principle:

> **The stronger the guarantees established at the boundary, the simpler the rest of the system can be.**

---

# 22. Constructors are part of the type's contract

Consider:

```kotlin
class Rectangle(
    val width: Double,
    val height: Double
)
```

When you see:

```kotlin
Rectangle(10.0, 5.0)
```

you know a Rectangle requires:

```text
width
height
```

That's part of the type's interface.

Later, when you see a more complex class:

```kotlin
User(
    id = ...,
    name = ...,
    email = ...
)
```

the constructor tells you what is required to create a valid object.

This is one reason well-designed constructors matter.

---

# 23. Secondary constructors

Kotlin also supports another form of constructor.

For example:

```kotlin
class Person(
    val name: String,
    val age: Int
) {
    constructor(name: String) : this(name, 0)
}
```

Now both work:

```kotlin
val p1 = Person("Alice", 25)
val p2 = Person("Bob")
```

The second constructor delegates to the primary constructor:

```text
Person("Bob")
   ↓
Person("Bob", 0)
```

Secondary constructors are useful in some situations, especially interoperability scenarios.

But in idiomatic Kotlin, you can often accomplish the same thing more simply with:

- default arguments,
    
- factory functions,
    
- named constructors through companion objects.
    

We'll discuss those alternatives later.

---

# 24. Default constructor values can be simpler

Instead of:

```kotlin
class Person(
    val name: String,
    val age: Int
) {
    constructor(name: String) : this(name, 0)
}
```

you can often write:

```kotlin
class Person(
    val name: String,
    val age: Int = 0
)
```

Now:

```kotlin
Person("Bob")
```

works automatically.

This is another place where Kotlin's default arguments reduce boilerplate.

---

# 25. Object identity

Now let's look at a subtle concept.

Suppose:

```kotlin
val person1 = Person("Alice", 25)
val person2 = Person("Alice", 25)
```

They contain identical data.

But they are still **two separate objects**.

Conceptually:

```text
person1 ──→ Object A
             name = Alice
             age = 25

person2 ──→ Object B
             name = Alice
             age = 25
```

So:

```text
same data
≠
same object
```

This will become very important when we study equality.

---

# 26. `==` vs `===`

Remember our earlier lesson?

```text
==   → equality
===  → referential identity
```

For ordinary classes, `==` eventually delegates to equality semantics defined for the objects, while `===` asks whether two references point to the exact same object instance.

For example:

```kotlin
val person1 = Person("Alice", 25)
val person2 = Person("Alice", 25)

println(person1 === person2)
```

These are two separately created objects, so the result is:

```text
false
```

And for a normal class that hasn't customized equality:

```kotlin
println(person1 == person2)
```

will also not consider them equal just because their fields happen to match.

This behavior is exactly why Kotlin has a specialized `data class`, which we'll learn next.

Don't worry if equality still feels incomplete here.

It is.

That's intentional.

---

# 27. Class properties and object state

Let's revisit our account:

```kotlin
class BankAccount(
    val owner: String,
    var balance: Double
) {
    fun deposit(amount: Double) {
        balance += amount
    }
}
```

Consider:

```kotlin
val aliceAccount = BankAccount("Alice", 1000.0)
val bobAccount = BankAccount("Bob", 500.0)
```

There are now two separate pieces of state:

```text
Alice account
    owner = Alice
    balance = 1000

Bob account
    owner = Bob
    balance = 500
```

Calling:

```kotlin
aliceAccount.deposit(200.0)
```

changes Alice's account.

It doesn't change Bob's.

This gives us a very important mental model:

> **An object is a value/entity with its own state and behavior.**

---

# 28. Why classes matter for modeling

Imagine a real application dealing with an online store.

You might need:

```text
Product
Customer
Order
Address
Payment
Cart
```

Each has different information and behavior.

Without custom types, everything might collapse into:

```text
Strings
Ints
Doubles
Lists
Maps
```

and your code would have to constantly remember which value means what.

With classes, you can represent the concepts directly:

```kotlin
class Product(...)
class Customer(...)
class Order(...)
class Address(...)
```

Now your program begins to resemble the domain it models.

That's one of the great strengths of object-oriented modeling.

---

# 29. But classes aren't just "containers of variables"

This is an important warning.

You might see:

```kotlin
class Person(
    val name: String,
    val age: Int
)
```

and conclude:

> A class is just a group of variables.

That's incomplete.

A class can contain:

- state,
    
- behavior,
    
- validation,
    
- invariants,
    
- relationships,
    
- computed properties,
    
- visibility rules,
    
- lifecycle logic.
    

For example:

```kotlin
class BankAccount(
    val owner: String,
    initialBalance: Double
) {
    private var balance = initialBalance

    fun deposit(amount: Double) {
        require(amount > 0)
        balance += amount
    }

    fun withdraw(amount: Double) {
        require(amount > 0)
        require(amount <= balance)

        balance -= amount
    }

    fun currentBalance(): Double {
        return balance
    }
}
```

Now the class isn't merely holding data.

It's enforcing rules around that data.

That's much closer to the power of object-oriented design.

---

# 30. Notice how the constructor can hide implementation details

We wrote:

```kotlin
class BankAccount(
    val owner: String,
    initialBalance: Double
)
```

Notice:

```text
initialBalance
```

doesn't have `val` or `var`.

So it's a constructor parameter, not automatically a public property.

Then:

```kotlin
private var balance = initialBalance
```

stores it internally.

Outside code can't directly manipulate the balance.

This is a nice example of separating:

```text
input required to construct an object
```

from:

```text
state the object chooses to expose
```

That distinction becomes very important in good API design.

---

# 31. Your first real class

Try writing this:

```kotlin
class Rectangle(
    val width: Double,
    val height: Double
) {
    val area: Double
        get() = width * height

    val perimeter: Double
        get() = 2 * (width + height)
}
```

Then:

```kotlin
fun main() {
    val rectangle = Rectangle(10.0, 5.0)

    println(rectangle.area)
    println(rectangle.perimeter)
}
```

You should get:

```text
50.0
30.0
```

Notice what you've created:

```text
Rectangle
│
├── width
├── height
├── area
└── perimeter
```

You have effectively created a domain-specific type.

---

# 32. Now add behavior

Let's create a bank account:

```kotlin
class BankAccount(
    val owner: String,
    initialBalance: Double
) {
    private var balance = initialBalance

    fun deposit(amount: Double) {
        require(amount > 0)
        balance += amount
    }

    fun withdraw(amount: Double) {
        require(amount > 0)
        require(amount <= balance)

        balance -= amount
    }

    fun getBalance(): Double {
        return balance
    }
}
```

Then:

```kotlin
fun main() {
    val account = BankAccount("Alice", 1000.0)

    account.deposit(500.0)
    account.withdraw(200.0)

    println(account.getBalance())
}
```

Output:

```text
1300.0
```

Now you're beginning to build software around **objects that own and control their state**.

---

# 33. One design question

Why did we make:

```kotlin
private var balance
```

instead of:

```kotlin
var balance
```

?

Because we don't necessarily want arbitrary external code to do:

```kotlin
account.balance = -999999.0
```

We want operations such as:

```kotlin
account.deposit(...)
account.withdraw(...)
```

to enforce the rules.

This is the essence of encapsulation.

---

# 34. Your exercises

### Exercise 1 — Person

Create:

```kotlin
class Person(
    val name: String,
    var age: Int
)
```

Then create two different `Person` objects and modify one person's age.

Verify that the other person's age doesn't change.

---

### Exercise 2 — Rectangle

Create the `Rectangle` class from above.

Add a property:

```text
isSquare
```

which is `true` when width and height are equal.

Try doing it as a computed property.

---

### Exercise 3 — Bank account

Create a `BankAccount` class with:

```text
owner
private balance
deposit()
withdraw()
currentBalance()
```

Add validation so:

- deposit amount must be positive,
    
- withdrawal amount must be positive,
    
- withdrawal cannot exceed balance.
    

---

### Exercise 4 — Constructor parameters

Create a class like:

```kotlin
class User(
    name: String
)
```

Notice that `name` isn't a property.

Then change it to:

```kotlin
class User(
    val name: String
)
```

and observe the difference.

This is a tiny exercise, but understanding it properly matters.

---

# Challenge

Design a `ShoppingCart` class.

It should conceptually have:

```text
items
addItem(...)
removeItem(...)
totalPrice
```

Don't worry about advanced collections yet.

You can simplify it to a list of prices:

```kotlin
class ShoppingCart {
    ...
}
```

Try to decide:

> What state should the object own?

> Which state should be private?

> Which operations should be exposed?

> Should `totalPrice` be stored, or calculated from the items?

That last question is especially interesting.

You might choose:

```text
stored total
```

or:

```text
computed total
```

Both have consequences.

Don't worry about making the perfect design. Think about the trade-offs.

---

# The mental model I want you to have now

We've moved from:

```text
primitive values
```

to:

```text
custom types
```

A class defines a type:

```text
class Person
```

An object is an instance:

```text
Person("Alice", 25)
```

And an object can contain:

```text
state
+
behavior
```

So:

```text
                 Class
                  │
          ┌───────┴───────┐
          ↓               ↓
        State           Behavior
          ↓               ↓
     properties        functions
          │
          └───────┬───────┘
                  ↓
                Object
```

And encapsulation lets us control how the outside world interacts with that state.

---

There's one thing we haven't addressed yet that becomes **very important immediately**:

Our `Person` objects with the same data aren't automatically considered equal.

For example:

```kotlin
Person("Alice", 25) == Person("Alice", 25)
```

raises the deeper question:

> **What does it mean for two objects to be "the same"?**

Do we care that they are the exact same object?

Or do we care that they contain the same information?

Kotlin has a particularly elegant feature for data-oriented objects that answers much of this:

# **`data class`**

That's our next lesson, where we'll explore `data class`, `equals`, `hashCode`, `toString`, `copy`, destructuring, and why data classes are so important in real Kotlin code.