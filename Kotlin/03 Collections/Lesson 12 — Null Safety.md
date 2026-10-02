
# Lesson 12 — Null Safety: `null`, `?`, `?.`, and `?:`

This is one of the lessons I want you to **really understand**, rather than just memorize a handful of operators.

You've already encountered something interesting with maps:

```kotlin
val ages = mapOf(
    "Alice" to 25,
    "Bob" to 30
)

println(ages["Charlie"])
```

What happens when `"Charlie"` isn't there?

There isn't an age to return.

The result is:

```text
null
```

And this creates an important question:

> What exactly does `null` mean, and why does Kotlin make such a big deal about it?

Kotlin's type system explicitly distinguishes nullable and non-nullable types, so many possible null-related mistakes are caught by the compiler rather than waiting until the program runs. ([Kotlin](https://kotlinlang.org/docs/null-safety.html?utm_source=chatgpt.com "Null safety | Kotlin Documentation"))

---

# 1. What is `null`?

`null` essentially represents:

> **There is no value here.**

Suppose you have a person whose middle name isn't known.

You might have:

```text
name = "Alice"
middleName = null
```

The important idea is that `null` isn't a string.

It's not:

```text
""
```

and it's not:

```text
"null"
```

Those are completely different things.

```text
""       → an empty String
"null"   → a String containing four characters
null     → absence of a value
```

That distinction causes a huge number of bugs in software.

---

# 2. Kotlin makes you explicitly say when `null` is allowed

This is the central feature of Kotlin's null-safety system.

Consider:

```kotlin
val name: String = "Alice"
```

`name` is a `String`.

Kotlin assumes:

> This variable will contain an actual String, not `null`.

Therefore:

```kotlin
val name: String = null
```

is rejected by the compiler. ([Kotlin](https://kotlinlang.org/docs/null-safety.html?utm_source=chatgpt.com "Null safety | Kotlin Documentation"))

Now suppose `null` genuinely is a valid possibility.

You explicitly mark the type as nullable:

```kotlin
val name: String? = null
```

That `?` means:

> This is a `String`, or it may be `null`.

So:

```text
String
```

means:

```text
String only
```

while:

```text
String?
```

means:

```text
String OR null
```

This tiny symbol is one of the most important symbols in Kotlin.

---

# 3. `String` and `String?` are different types

This is worth taking seriously.

```kotlin
val a: String = "Hello"
val b: String? = "Hello"
```

They both currently contain `"Hello"`.

But their **types are different**.

```text
a → String
b → String?
```

Why?

Because `b` is allowed to later become:

```kotlin
b = null
```

whereas `a` cannot.

This is why Kotlin needs to treat them differently.

The question isn't merely:

> What's the value right now?

It's also:

> **Could this value be null according to its type?**

---

# 4. The compiler uses that information

Look at:

```kotlin
val name: String = "Alice"

println(name.length)
```

This is completely fine.

Kotlin knows:

```text
name : String
```

and a `String` isn't allowed to be `null`.

So accessing:

```kotlin
name.length
```

is safe from the particular problem of `name` being null.

Now:

```kotlin
val name: String? = "Alice"

println(name.length)
```

Kotlin rejects this.

Why?

Because from the type's perspective:

```text
name : String?
```

means:

```text
name could be "Alice"
OR
name could be null
```

If it were null, what would:

```kotlin
name.length
```

mean?

There would be no String whose length we could retrieve.

So Kotlin makes you deal with the possibility explicitly. ([Kotlin](https://kotlinlang.org/docs/null-safety.html?utm_source=chatgpt.com "Null safety | Kotlin Documentation"))

---

# 5. This is what Kotlin is protecting you from

In many languages, you can accidentally do something conceptually like:

```text
someReference = null
someReference.doSomething()
```

and only discover the problem when the program crashes.

That family of failures is commonly associated with `NullPointerException`, or NPE.

Kotlin's design moves a large portion of these mistakes into **compile-time checking** instead. ([Kotlin](https://kotlinlang.org/docs/null-safety.html?utm_source=chatgpt.com "Null safety | Kotlin Documentation"))

The compiler is essentially saying:

> "You've admitted that this value may be absent. Show me what you want to do in that situation."

That's a very powerful idea.

---

# 6. The first way to handle nullable values: check with `if`

Suppose:

```kotlin
val name: String? = "Alice"
```

We can explicitly check:

```kotlin
if (name != null) {
    println(name.length)
}
```

Now Kotlin knows that **inside this branch**, `name` is not null.

This is called a **smart cast**.

Conceptually:

```text
name : String?

check:
name != null

inside branch:
name can safely be treated as String
```

So:

```kotlin
if (name != null) {
    println(name.length)
}
```

works.

Kotlin tracks the null check and can narrow the type in appropriate situations. ([Kotlin](https://kotlinlang.org/docs/null-safety.html?utm_source=chatgpt.com "Null safety | Kotlin Documentation"))

---

# 7. Why does this work?

Suppose:

```kotlin
val name: String? = "Alice"
```

At the start, Kotlin has to consider:

```text
String
or
null
```

Then you write:

```kotlin
if (name != null) {
```

Inside that branch, you've established:

```text
name ≠ null
```

Therefore, for the relevant flow of execution:

```text
String? 
   ↓
after null check
   ↓
String
```

This is one of the first examples of the compiler using **control-flow information** to reason about types.

That idea will appear again with other Kotlin type-system features.

---

# 8. The safe-call operator `?.`

The explicit `if` is useful, but Kotlin has a much more compact way to express the same general idea.

Suppose:

```kotlin
val name: String? = "Alice"
```

Instead of:

```kotlin
if (name != null) {
    println(name.length)
}
```

we can write:

```kotlin
println(name?.length)
```

Read:

```text
name?.length
```

as:

> "If `name` isn't null, get its length. Otherwise, give me null."

So:

```kotlin
val name: String? = "Alice"

println(name?.length)
```

prints:

```text
5
```

But:

```kotlin
val name: String? = null

println(name?.length)
```

prints:

```text
null
```

The safe-call operator is a very common part of Kotlin. ([Kotlin](https://kotlinlang.org/docs/null-safety.html?utm_source=chatgpt.com "Null safety | Kotlin Documentation"))

---

# 9. What does `?.` actually do?

Suppose:

```kotlin
val name: String? = null
```

Then:

```kotlin
name?.length
```

conceptually behaves like:

```kotlin
if (name != null) {
    name.length
} else {
    null
}
```

So:

```text
name = "Alice"
        ↓
name?.length
        ↓
5
```

but:

```text
name = null
        ↓
name?.length
        ↓
null
```

This is why it's called a **safe call**.

It doesn't attempt to access the member when the receiver is null.

---

# 10. Notice that the result can itself be nullable

If:

```kotlin
val name: String? = "Alice"
```

then:

```kotlin
val length = name?.length
```

gives you an:

```text
Int?
```

not an ordinary:

```text
Int
```

Why?

Because:

```text
name could be non-null
    → length is an Int

name could be null
    → result is null
```

Therefore:

```text
name?.length : Int?
```

This is an important pattern:

> **Nullable input can produce nullable output.**

We'll see this repeatedly.

---

# 11. Chaining safe calls

This becomes extremely useful with nested objects.

Imagine:

```text
person
  ↓
department
  ↓
manager
  ↓
name
```

Any of those could potentially be absent.

You could write:

```kotlin
person?.department?.manager?.name
```

This means:

> Start with `person`; if it's non-null, access `department`; if that's non-null, access `manager`; if that's non-null, access `name`; otherwise return `null`.

Kotlin's official documentation specifically describes this chaining behavior. ([Kotlin](https://kotlinlang.org/docs/null-safety.html?utm_source=chatgpt.com "Null safety | Kotlin Documentation"))

Conceptually:

```text
person?
   ↓
department?
   ↓
manager?
   ↓
name
```

The whole expression safely stops at the first `null`.

---

# 12. Why `?.` is better than a mountain of `if`s

Without safe calls, you might end up with:

```kotlin
if (person != null) {
    if (person.department != null) {
        if (person.department.manager != null) {
            println(person.department.manager.name)
        }
    }
}
```

That's hard to read.

Using safe calls:

```kotlin
println(person?.department?.manager?.name)
```

expresses the same overall idea much more directly.

That doesn't mean `?.` should replace every `if`.

Sometimes you genuinely need different behavior depending on which condition failed.

But when the desired behavior is simply:

> "Give me the value if the chain exists, otherwise null"

safe calls are excellent.

---

# 13. The Elvis operator `?:`

Now suppose you don't want the result to remain `null`.

Suppose:

```kotlin
val name: String? = null
```

and you want:

> Use the actual name if available; otherwise use `"Unknown"`.

You can write:

```kotlin
val displayName = name ?: "Unknown"
```

This is called the **Elvis operator**.

Read it as:

> If the thing on the left is not null, use it; otherwise use the thing on the right.

So:

```text
"Alice" ?: "Unknown"
     ↓
"Alice"
```

while:

```text
null ?: "Unknown"
     ↓
"Unknown"
```

Kotlin's documentation describes `?:` exactly in this role. ([Kotlin](https://kotlinlang.org/docs/null-safety.html?utm_source=chatgpt.com "Null safety | Kotlin Documentation"))

---

# 14. `?.` and `?:` work beautifully together

Suppose:

```kotlin
val name: String? = null
```

You could write:

```kotlin
val length = name?.length ?: 0
```

Read it in two stages.

First:

```text
name?.length
```

means:

```text
length if name exists
otherwise null
```

Then:

```text
?: 0
```

means:

```text
if the result was null, use 0
```

So:

```text
name = "Alice"
    ↓
name?.length
    ↓
5
    ↓
5 ?: 0
    ↓
5
```

But:

```text
name = null
    ↓
name?.length
    ↓
null
    ↓
null ?: 0
    ↓
0
```

This pattern is everywhere in Kotlin.

---

# 15. You can think of `?:` as a fallback

Some examples:

```kotlin
val username = inputName ?: "Guest"
```

```kotlin
val timeout = configuredTimeout ?: 30
```

```kotlin
val title = userTitle ?: "No title"
```

The pattern is:

```text
preferred value
      ↓
is it null?
   /      \
 no        yes
 ↓          ↓
use it   use fallback
```

It's not only a null-handling operator.

It's an expression for:

> **Use this value, otherwise use this alternative.**

---

# 16. `if` vs `?.` vs `?:`

Suppose:

```kotlin
val name: String? = ...
```

These are different tools.

### `if`

Use when you need explicit control:

```kotlin
if (name != null) {
    println("Name exists")
} else {
    println("No name")
}
```

### `?.`

Use when you want to safely continue an operation:

```kotlin
println(name?.length)
```

### `?:`

Use when you want a fallback:

```kotlin
val length = name?.length ?: 0
```

This distinction will become second nature with practice.

---

# 17. Now meet `!!`

Kotlin gives you one way to essentially tell the compiler:

> "Trust me. I know this isn't null."

That's:

```kotlin
!!
```

For example:

```kotlin
val name: String? = "Alice"

println(name!!.length)
```

This works because the value is non-null.

But now:

```kotlin
val name: String? = null

println(name!!.length)
```

causes a `NullPointerException`. Kotlin's documentation specifically identifies `!!` as a way an NPE can still be triggered. ([Kotlin](https://kotlinlang.org/docs/null-safety.html?utm_source=chatgpt.com "Null safety | Kotlin Documentation"))

---

# 18. Why does `!!` exist?

Because sometimes the programmer knows something the compiler cannot prove.

For example, some complicated initialization or interoperability situation may make it impossible for the compiler to establish non-nullness.

In such a case, you can explicitly assert:

```kotlin
value!!
```

But think of `!!` as:

> **"I am taking responsibility for this assumption."**

You're overriding the safety guarantee.

That's why I don't want you developing the habit:

```text
compiler complains about ?
      ↓
add !!
      ↓
problem solved
```

No.

Usually the compiler is pointing out a genuine design question:

> What should happen if this is null?

You should answer that question rather than simply silence the compiler.

---

# 19. A useful example of bad `!!`

Imagine:

```kotlin
val user = findUser("Alice")!!
```

If `findUser` doesn't find Alice, the program crashes.

Compare:

```kotlin
val user = findUser("Alice")

if (user != null) {
    println(user.name)
}
```

Now you've explicitly handled the missing-user case.

Or perhaps:

```kotlin
val user = findUser("Alice") ?: return
```

depending on what the surrounding function should do.

The correct choice depends on the business logic.

That's a much better way to think about null safety:

> `null` is not an annoying technical problem. It can represent a legitimate state in the domain.

---

# 20. `null` can mean different things

Suppose:

```kotlin
val middleName: String? = null
```

Maybe it means:

> This person genuinely has no middle name.

Or:

```kotlin
val searchResult: User? = null
```

might mean:

> No matching user was found.

Or:

```kotlin
val cachedData: Data? = null
```

might mean:

> We haven't loaded/cached the data yet.

Or:

```kotlin
val selectedItem: Item? = null
```

might mean:

> Nothing is currently selected.

The type system tells you:

```text
this thing may be absent
```

but **your domain determines what that absence means**.

That's a very important distinction.

---

# 21. `null` is not necessarily an error

This is a subtle but important mindset shift.

For example:

```kotlin
val numbers = listOf(10, 20, 30)

val result = numbers.find { it > 100 }
```

What should happen?

There is no matching element.

Returning `null` is perfectly sensible.

The absence of a result isn't necessarily a program failure.

That's one reason nullable return types are useful:

```text
User?
```

can naturally mean:

> A user may or may not exist.

We'll later learn other modeling approaches such as sealed result types, but nullable values are useful in many situations.

---

# 22. Nullable function parameters

A function can explicitly say it accepts null:

```kotlin
fun printName(name: String?) {
    println(name)
}
```

Then both work:

```kotlin
printName("Alice")
printName(null)
```

Inside the function, you need to respect the possibility of null.

For example:

```kotlin
fun printName(name: String?) {
    println(name?.uppercase())
}
```

If name is `"Alice"`:

```text
ALICE
```

If name is `null`:

```text
null
```

The function's signature communicates something important to its caller:

> "You are allowed to give me no name."

That's part of the function's contract.

---

# 23. Nullable return values

A function can return a nullable type too.

For example:

```kotlin
fun findUser(id: Int): String? {
    if (id == 1) {
        return "Alice"
    }

    return null
}
```

Now:

```kotlin
val user = findUser(1)
```

gives a nullable String.

And:

```kotlin
val user = findUser(999)
```

may produce `null`.

The caller must account for that.

This is a very powerful communication mechanism:

```text
String
→ caller can rely on a String

String?
→ caller must consider absence
```

---

# 24. Nullability propagates through expressions

Suppose:

```kotlin
val user: String? = ...
```

Then:

```kotlin
user?.length
```

is:

```text
Int?
```

And:

```kotlin
user?.uppercase()
```

is:

```text
String?
```

because the operation may or may not produce a value depending on whether `user` exists.

You can then continue:

```kotlin
user?.uppercase()?.length
```

Now the type is again nullable:

```text
Int?
```

This is one of the reasons the `?` notation is so useful.

The type system carries information about the possibility of absence through the program.

---

# 25. Null safety with collections

Remember:

```kotlin
val names = listOf("Alice", "Bob", "Charlie")
```

Suppose:

```kotlin
val result = names.find { it.startsWith("Z") }
```

`result` may be null.

We can safely write:

```kotlin
println(result?.uppercase())
```

Or:

```kotlin
val name = result ?: "Nobody"
```

Now the result is guaranteed to have a String value.

And there's another useful operation:

```kotlin
val values: List<String?> = listOf("A", null, "B", null, "C")

val nonNullValues = values.filterNotNull()
```

which gives:

```text
[A, B, C]
```

Kotlin provides `filterNotNull()` specifically for filtering nullable collection elements into a non-null collection. ([Kotlin](https://kotlinlang.org/docs/null-safety.html?utm_source=chatgpt.com "Null safety | Kotlin Documentation"))

---

# 26. `let` and nullability

You've already seen `let` indirectly in the curriculum.

A common pattern is:

```kotlin
val name: String? = ...
name?.let {
    println(it.length)
}
```

Read this as:

> If `name` is not null, execute this block with the non-null value.

The `it` is the non-null value inside the lambda.

Conceptually:

```text
name
 ↓
is null?
 /     \
yes     no
 ↓       ↓
nothing  let(value)
```

So:

```kotlin
name?.let { println(it) }
```

is another way to perform an action only when the value exists.

Kotlin documents this combination as a standard null-handling technique. ([Kotlin](https://kotlinlang.org/docs/null-safety.html?utm_source=chatgpt.com "Null safety | Kotlin Documentation"))

---

# 27. Don't turn every null check into `let`

You'll often see code like:

```kotlin
user?.let {
    println(it.name)
}
```

That's fine.

But don't automatically turn this:

```kotlin
if (user != null) {
    println(user.name)
}
```

into:

```kotlin
user?.let {
    println(it.name)
}
```

just because it looks more "Kotlin-like."

Sometimes the `if` version communicates the logic more clearly.

For example:

```kotlin
if (user == null) {
    println("User not found")
    return
}

println(user.name)
```

is arguably much clearer than trying to force everything into scope functions.

Again:

> Kotlin idioms are tools, not laws.

---

# 28. Early return with `?:`

Here's a nice Kotlin pattern.

Suppose:

```kotlin
fun printUserName(user: User?): String {
    val actualUser = user ?: return "No user"
    return actualUser.name
}
```

The Elvis operator can work with `return` because `return` is an expression in Kotlin. ([Kotlin](https://kotlinlang.org/docs/null-safety.html?utm_source=chatgpt.com "Null safety | Kotlin Documentation"))

Read:

```kotlin
val actualUser = user ?: return "No user"
```

as:

> If `user` exists, put it into `actualUser`; otherwise exit the function immediately.

This can make validation code very clean.

We'll see this pattern again when writing real applications.

---

# 29. A practical example

Imagine a shopping cart.

```kotlin
fun getDiscountCode(): String? {
    // Maybe there is no code.
    return null
}
```

Then:

```kotlin
val code = getDiscountCode()

val displayCode = code ?: "No discount code"

println(displayCode)
```

Now the absence of a discount code is handled explicitly.

Alternatively:

```kotlin
println(code?.uppercase() ?: "NO CODE")
```

This says:

1. If the code exists, uppercase it.
    
2. Otherwise use `"NO CODE"`.
    

Notice how much meaning is expressed in a single line.

---

# 30. Let's revisit our map example

Remember:

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

The result is nullable because `"Charlie"` might not exist.

So:

```kotlin
println(age)
```

could print:

```text
null
```

But:

```kotlin
println(age?.plus(1))
```

will safely produce null when there is no age.

Or:

```kotlin
val actualAge = age ?: 0
```

gives you a fallback.

Now you can see how our previous collection lesson and this null-safety lesson connect naturally.

---

# 31. `as?` — safe casting

There's another nullable-related operator worth knowing.

Suppose:

```kotlin
val value: Any = "Hello"
```

Later we'll study `Any` properly, but it essentially allows a value of many different types.

A normal cast:

```kotlin
val number = value as Int
```

would fail because the value is actually a String.

A safe cast:

```kotlin
val number = value as? Int
```

instead gives:

```text
null
```

because the conversion isn't possible.

Kotlin documents `as?` as the safe-cast operator: instead of throwing on a failed cast, it returns `null`. ([Kotlin](https://kotlinlang.org/docs/null-safety.html?utm_source=chatgpt.com "Null safety | Kotlin Documentation"))

We'll properly study type casting later.

For now, remember:

```text
as
→ force the cast

as?
→ attempt the cast safely
```

---

# 32. A critical distinction: compile-time safety doesn't mean "null can never happen"

It's tempting to think:

> "Kotlin has null safety, therefore Kotlin programs cannot have NPEs."

That's too strong.

Kotlin significantly reduces null-related failures, but NPEs can still occur through things such as explicit `!!`, some initialization problems, and Java interoperability/platform types. ([Kotlin](https://kotlinlang.org/docs/null-safety.html?utm_source=chatgpt.com "Null safety | Kotlin Documentation"))

So the accurate idea is:

> **Kotlin's type system makes many null mistakes visible before runtime.**

That's different from guaranteeing that every possible null-related failure is impossible.

---

# 33. Why this design is powerful

Imagine a function:

```kotlin
fun getUserName(): String
```

The caller can reasonably expect:

```text
There is always a String result.
```

Whereas:

```kotlin
fun getUserName(): String?
```

communicates:

```text
A String might exist,
but there may be no value.
```

The type itself communicates part of the function's behavior.

That's a recurring Kotlin philosophy:

> **Put useful information into types so the compiler and humans can reason about the program.**

You'll see this idea again with:

- generic types,
    
- sealed classes,
    
- variance,
    
- value classes,
    
- result modeling.
    

---

# 34. Let's put everything together

Here's a small program:

```kotlin
fun findUserName(id: Int): String? {
    return if (id == 1) {
        "Alice"
    } else {
        null
    }
}

fun main() {
    val name = findUserName(2)

    val length = name?.length ?: 0

    println("Name: ${name ?: "Unknown"}")
    println("Length: $length")
}
```

For `id = 2`:

```text
findUserName(2)
       ↓
null

name?.length
       ↓
null

null ?: 0
       ↓
0

name ?: "Unknown"
       ↓
"Unknown"
```

Output:

```text
Name: Unknown
Length: 0
```

Now look at how many concepts you've combined:

```text
nullable return type
       ↓
null
       ↓
safe call
       ↓
Elvis fallback
       ↓
string template
```

That's real Kotlin.

---

# 35. Your exercises

### Exercise 1

Create:

```kotlin
var name: String? = "Alice"
```

Print its length using `?.`.

Then change it to:

```kotlin
name = null
```

and run it again.

Observe the result.

---

### Exercise 2

Create:

```kotlin
val name: String? = null
```

Use `?:` so that the program prints:

```text
Unknown
```

---

### Exercise 3

Write:

```text
findUser(id)
```

that returns a `String?`.

For one particular ID, return a name.

For everything else, return `null`.

Then safely use the result.

---

### Exercise 4

Given:

```kotlin
val ages = mapOf(
    "Alice" to 25,
    "Bob" to 30
)
```

retrieve the age of `"Charlie"`.

Use `?:` to give them an age of `0` when they don't exist.

Ask yourself whether `0` is actually a good domain fallback. That's a design question, not merely a Kotlin question.

---

### Exercise 5

Try deliberately writing code that the compiler rejects:

```kotlin
val name: String? = null
println(name.length)
```

Then fix it using:

```text
if
```

Then:

```text
?.
```

Then:

```text
?. + ?:
```

The point is to see that the same underlying problem can be expressed in several different ways.

---

# Challenge

Write:

```kotlin
fun getUsername(id: Int): String?
```

Then write a program that:

1. calls it,
    
2. prints the username in uppercase if it exists,
    
3. otherwise prints `"GUEST"`,
    
4. prints the length of the resulting displayed name.
    

Try to do it first using explicit `if`.

Then try using:

```text
?.
```

and:

```text
?:
```

After you've solved it, compare the versions and decide which one you find easiest to understand.

---

# The mental model to keep

This is the most important thing from today's lesson:

```text
String
  ↓
definitely a String

String?
  ↓
String OR null
```

And then:

```text
String?
   │
   ├── if (x != null)
   │       ↓
   │     safely use x
   │
   ├── x?.something
   │       ↓
   │     value or null
   │
   ├── x ?: fallback
   │       ↓
   │     value or fallback
   │
   └── x!!
           ↓
        "I guarantee
         this isn't null"
```

The first three are ways of **handling uncertainty**.

The last one is you saying:

> "I accept responsibility for this assumption."

That's why `!!` should be treated carefully.

And there's an even deeper idea underneath all of this:

> **Nullability is part of the type.**

That means a function doesn't merely tell you:

```text
"I'm returning a String."
```

It can tell you:

```text
"I'm returning a String, but there may be no String."
```

The compiler can then use that information to force you to deal with the possibility.

That is one of the defining ideas of Kotlin.

---

We've now covered a substantial part of the language's basic machinery:

**variables → types → expressions → control flow → functions → lambdas → collections → null safety.**

The next lesson will take us back to **strings and text**, but at a much deeper level: string templates, interpolation, multiline strings, common string operations, equality, and then we'll use those concepts to start building programs that work with actual textual input rather than hard-coded values.