
# Lesson 13 — Strings: Working with Text Properly

We've been using strings since the beginning:

```kotlin
val name = "Alice"
```

but so far we've mostly treated a `String` as:

> "some text."

That's not enough.

In real programs, text is everywhere:

```text
usernames
passwords
messages
file contents
URLs
JSON
database values
search queries
logs
HTML
names
addresses
```

So we need to understand what a string actually is, how Kotlin represents it, how to combine strings, search them, compare them, extract pieces, and deal with common problems.

There's also a subtle but important distinction between a **`Char`** and a **`String`**, which we'll start with.

---

# 1. `Char` vs `String`

You've already seen:

```kotlin
val letter = 'A'
```

and:

```kotlin
val word = "Alice"
```

They are different types.

```text
'A'       → Char
"Alice"   → String
```

A `Char` represents one character.

A `String` represents a sequence of characters.

For example:

```text
'A'
'B'
'7'
'?'
```

are individual characters.

While:

```text
"ABC"
"Hello"
"Kotlin"
```

are strings.

You can think of:

```text
String
  ↓
sequence of characters
```

So:

```text
"Kotlin"
 ↓
K  o  t  l  i  n
```

We'll eventually connect this to indexing and iteration.

---

# 2. Creating strings

The normal syntax is:

```kotlin
val name = "Alice"
```

Strings use **double quotes**.

You can have an empty string:

```kotlin
val empty = ""
```

That's a valid `String`.

And you can have spaces:

```kotlin
val message = "Hello Kotlin"
```

Spaces are characters too.

---

# 3. Strings can contain almost any ordinary text

For example:

```kotlin
val sentence = "Kotlin is fun"
val number = "12345"
val symbols = "!@#$%"
```

Notice this:

```kotlin
val number = "12345"
```

Even though it looks like a number, it's a **String**, not an `Int`.

So:

```text
12345
```

and:

```text
"12345"
```

are very different.

The first represents a numeric value.

The second represents text.

This matters enormously.

For example:

```kotlin
val a = 10
val b = 20

println(a + b)
```

produces:

```text
30
```

But:

```kotlin
val a = "10"
val b = "20"

println(a + b)
```

produces:

```text
1020
```

because you're combining text.

That is our first example of how **type determines meaning**.

---

# 4. String concatenation

You can combine strings using `+`.

```kotlin
val firstName = "Alice"
val lastName = "Smith"

val fullName = firstName + " " + lastName

println(fullName)
```

Output:

```text
Alice Smith
```

Conceptually:

```text
"Alice"
   +
" "
   +
"Smith"
   ↓
"Alice Smith"
```

This is called **string concatenation**.

---

# 5. But Kotlin gives us something nicer: string templates

Instead of:

```kotlin
val name = "Alice"
val age = 25

println("Name: " + name + ", Age: " + age)
```

you can write:

```kotlin
println("Name: $name, Age: $age")
```

This is a **string template**.

The `$name` means:

> Put the value of `name` into this string.

So:

```kotlin
val name = "Alice"
println("Hello, $name")
```

produces:

```text
Hello, Alice
```

This is one of the most commonly used Kotlin features.

---

# 6. Expressions inside string templates

A simple variable can use:

```kotlin
$name
```

But for a more complicated expression, use `${...}`.

For example:

```kotlin
val age = 20

println("Next year I will be ${age + 1}")
```

Output:

```text
Next year I will be 21
```

The expression:

```kotlin
age + 1
```

gets evaluated first.

Then its result becomes part of the string.

Conceptually:

```text
age + 1
   ↓
21
   ↓
"Next year I will be 21"
```

So string templates connect directly to our previous lesson:

> **Expressions produce values.**

Those values can be inserted into strings.

---

# 7. Why `${}` exists

This:

```kotlin
"$name"
```

is enough when you're simply inserting a variable.

But imagine:

```kotlin
"$name.length"
```

What do you mean?

Do you mean:

```text
(name.length)
```

or:

```text
name + ".length"
```

Kotlin uses braces to make the expression boundary explicit:

```kotlin
"${name.length}"
```

So:

```kotlin
val name = "Alice"

println("${name.length}")
```

prints:

```text
5
```

And:

```kotlin
println("$name.length")
```

would be interpreted differently.

When in doubt, `${...}` makes the intended expression obvious.

---

# 8. You can call functions inside string templates

For example:

```kotlin
val name = "alice"

println("Uppercase: ${name.uppercase()}")
```

Output:

```text
Uppercase: ALICE
```

Or:

```kotlin
println("Length: ${name.length}")
```

Output:

```text
Length: 5
```

So a string template can contain an expression, and that expression can itself involve function calls.

This makes string formatting extremely expressive.

---

# 9. Strings are objects with useful operations

A `String` isn't just a box of characters.

It provides operations.

For example:

```kotlin
val name = "Kotlin"

println(name.length)
println(name.uppercase())
println(name.lowercase())
```

Output:

```text
6
KOTLIN
kotlin
```

Here we've already encountered three different operations:

```text
length
uppercase()
lowercase()
```

Notice an interesting syntax difference.

```kotlin
name.length
```

has no parentheses.

```kotlin
name.uppercase()
```

does.

Why?

Because:

```text
length
→ property

uppercase()
→ function call
```

We'll study properties and methods properly when we reach classes and objects.

For now, just recognize the distinction.

---

# 10. Indexing into a string

A string is a sequence of characters, so you can access a character by index.

```kotlin
val word = "Kotlin"

println(word[0])
println(word[1])
println(word[5])
```

Output:

```text
K
o
n
```

Remember: indexing starts at `0`.

So:

```text
index:  0 1 2 3 4 5
value:  K o t l i n
```

Therefore:

```kotlin
word[0]
```

is `'K'`.

And:

```kotlin
word[5]
```

is `'n'`.

The result of indexing a string is a `Char`.

So:

```kotlin
val first = word[0]
```

gives:

```text
first : Char
```

---

# 11. What happens if you use an invalid index?

For example:

```kotlin
val word = "Kotlin"

println(word[10])
```

There is no character at index `10`.

That leads to a runtime exception.

This introduces an important distinction:

```text
type safety
```

doesn't mean:

> "Every operation you write is logically valid."

The compiler can know:

```text
word is a String
```

but it may not know what index you'll provide at runtime.

So some errors can still occur during execution.

We'll later study exceptions properly.

For now, the lesson is:

> **A valid type doesn't guarantee that every operation on that type is safe for every possible input.**

---

# 12. Iterating over characters

Remember our `for` loop?

It works with strings:

```kotlin
val word = "Kotlin"

for (character in word) {
    println(character)
}
```

Output:

```text
K
o
t
l
i
n
```

This is another important connection.

A string can be thought of as a sequence of characters, and therefore we can iterate over it.

We don't need to manually write:

```kotlin
println(word[0])
println(word[1])
...
```

We can simply say:

> For each character in the string...

This is exactly the general idea behind iteration we discussed earlier.

---

# 13. Checking whether text contains something

Suppose:

```kotlin
val message = "Kotlin is powerful"
```

You can ask:

```kotlin
println("Kotlin" in message)
```

which produces:

```text
true
```

And:

```kotlin
println("Java" in message)
```

produces:

```text
false
```

You can also use:

```kotlin
message.contains("Kotlin")
```

The same general idea:

```text
Does this text contain this other text?
```

This is a common operation in search and validation code.

---

# 14. Checking the beginning and end

You can ask whether a string begins with something:

```kotlin
val filename = "report.pdf"

println(filename.startsWith("report"))
```

→ `true`

And whether it ends with something:

```kotlin
println(filename.endsWith(".pdf"))
```

→ `true`

These operations are often useful when dealing with:

- filenames
    
- URLs
    
- commands
    
- identifiers
    
- user input
    

---

# 15. Searching for a substring

Suppose:

```kotlin
val text = "Kotlin programming"
```

You can use:

```kotlin
println(text.indexOf("programming"))
```

The result is the index at which the substring begins.

You'll get:

```text
7
```

because:

```text
K o t l i n _ p r o g r a m m i n g
0 1 2 3 4 5 6 7 ...
```

If the substring isn't found, `indexOf` returns `-1`.

That is another nice real-world example of why some functions return nullable values while others use sentinel values such as `-1`.

Different APIs make different design choices.

Kotlin's standard library has a large family of string-search operations.

We'll learn the important ones rather than memorizing every API.

---

# 16. Extracting part of a string

Suppose:

```kotlin
val text = "Kotlin"
```

You can use:

```kotlin
println(text.substring(0, 3))
```

which gives:

```text
Kot
```

The range begins at `0` and ends **before** `3`.

So:

```text
indices:
0 1 2 3 4 5
K o t l i n
```

`substring(0, 3)` gives:

```text
K o t
```

This "start inclusive, end exclusive" pattern is very common in programming.

We'll encounter it in other contexts too.

---

# 17. But be careful with substring boundaries

Suppose:

```kotlin
val text = "Kotlin"
```

and you ask for:

```kotlin
text.substring(0, 20)
```

That isn't valid because the requested range extends beyond the string.

Again:

```text
compiler checks types
```

but:

```text
runtime checks many value-specific constraints
```

The compiler knows this:

```text
text : String
```

It may not know the exact runtime bounds of your indexes.

That's why both compile-time and runtime correctness matter.

---

# 18. Removing surrounding whitespace

User input frequently contains accidental spaces.

For example:

```text
"   Alice   "
```

Kotlin provides:

```kotlin
val name = "   Alice   "

println(name.trim())
```

Result:

```text
Alice
```

`trim()` removes whitespace from the beginning and end.

This is extremely common in input processing.

---

# 19. Replacing text

For example:

```kotlin
val message = "I like Java"

val updated = message.replace("Java", "Kotlin")

println(updated)
```

Output:

```text
I like Kotlin
```

Notice that strings are typically treated as values; operations like `replace` give you a new string rather than changing the original string in place.

So:

```kotlin
val message = "I like Java"
val updated = message.replace("Java", "Kotlin")
```

means:

```text
message → "I like Java"
updated → "I like Kotlin"
```

The original `message` remains unchanged.

This is an important property of strings that we'll discuss more in the context of immutability.

---

# 20. Strings are immutable

This is an important concept.

Consider:

```kotlin
val text = "Hello"
```

You can't modify an individual character of the existing string like this:

```kotlin
text[0] = 'J'
```

That's not allowed.

Instead, you create another string:

```kotlin
val newText = "J" + text.substring(1)
```

or use an appropriate string operation.

The broader idea is:

> **A `String` is immutable.**

An operation like:

```kotlin
text.uppercase()
```

doesn't mutate the existing string.

It produces a new string value.

For example:

```kotlin
val text = "hello"
val upper = text.uppercase()

println(text)
println(upper)
```

Output:

```text
hello
HELLO
```

So:

```text
text  → "hello"
upper → "HELLO"
```

This is a valuable model when reasoning about text.

---

# 21. Why immutability is useful

If strings could arbitrarily mutate while other parts of your application were using them, reasoning about them would become more complicated.

Instead:

```text
"hello"
```

stays `"hello"`.

If you need a different version:

```text
"HELLO"
```

you get another value.

This makes strings predictable.

We'll later discover that immutability can also be a very powerful tool when dealing with concurrency and shared data.

---

# 22. Escape characters

What if you actually want quotation marks inside a string?

For example:

```text
He said "Hello".
```

You can escape the quotation marks:

```kotlin
val message = "He said \"Hello\"."
```

The `\` tells Kotlin that the quote is part of the string rather than the end of the string.

Some common escape sequences are:

```text
\n  newline
\t  tab
\"  double quote
\\  backslash
```

For example:

```kotlin
println("Hello\nWorld")
```

produces:

```text
Hello
World
```

And:

```kotlin
println("A\tB\tC")
```

contains tab characters between the letters.

---

# 23. Triple-quoted strings

Kotlin also has **raw strings** using triple quotes:

```kotlin
val text = """
    Hello
    This is a
    multiline string.
""".trimIndent()
```

This is useful when you have:

- multiline text
    
- JSON
    
- SQL
    
- templates
    
- documentation
    
- text containing many quotes
    

You don't have to escape every quotation mark.

For example:

```kotlin
val json = """
    {
        "name": "Alice",
        "age": 25
    }
""".trimIndent()
```

That's much more readable than escaping every quote.

---

# 24. `trimIndent()`

Notice:

```kotlin
""".trimIndent()
```

Why is it there?

Our source code is indented:

```kotlin
val text = """
    Hello
    This is a
    multiline string.
""".trimIndent()
```

Without cleaning it up, the indentation from the source code would become part of the resulting text.

`trimIndent()` removes common indentation.

So it lets you format the Kotlin source nicely while producing cleaner text.

This is a good example of a recurring programming theme:

> **Source-code formatting and runtime data aren't always the same thing.**

---

# 25. Comparing strings

You can use:

```kotlin
val a = "Alice"
val b = "Alice"

println(a == b)
```

which produces:

```text
true
```

This checks whether their contents are equal.

You'll occasionally see:

```kotlin
a === b
```

That is a different concept: **referential identity**.

We aren't going to dive deeply into `===` yet because it becomes much more meaningful once we understand objects and references.

For ordinary string-content comparisons:

```kotlin
==
```

is what you generally want.

This is a good example of why learning operators in isolation isn't enough. Their full meaning becomes clearer as we learn Kotlin's object model.

---

# 26. Case sensitivity

String comparisons are normally case-sensitive.

For example:

```kotlin
println("Alice" == "alice")
```

produces:

```text
false
```

because:

```text
A ≠ a
```

Sometimes you specifically want case-insensitive comparison.

Kotlin provides APIs for that, but the important lesson is broader:

> **Text comparison depends on the exact comparison rules you're choosing.**

This becomes important in usernames, search, authentication, localization and user-facing applications.

We'll eventually discuss Unicode and locale-related issues as well.

---

# 27. Converting other values to strings

You can easily put other values into strings through templates:

```kotlin
val age = 25

val message = "Age: $age"
```

Kotlin converts the value into text for representation in that string.

You can also explicitly call:

```kotlin
age.toString()
```

For example:

```kotlin
val age = 25
val text = age.toString()
```

Now:

```text
age  → Int
text → String
```

This conversion is useful when an API specifically requires a `String`.

---

# 28. The opposite conversion

What if you have:

```kotlin
val text = "25"
```

and actually want the number `25`?

Now we need **parsing**.

You can write:

```kotlin
val number = text.toInt()
```

Now:

```text
text   → String
number → Int
```

But there's a danger.

What if:

```kotlin
val text = "hello"
```

and you do:

```kotlin
text.toInt()
```

That can't be converted into an integer.

An exception can occur.

So conversion and parsing are not the same thing.

This will become important when we start dealing with user input.

---

# 29. Safer parsing

Kotlin provides:

```kotlin
toIntOrNull()
```

For example:

```kotlin
val text = "25"

val number = text.toIntOrNull()
```

If the input is valid:

```text
number → 25
```

If:

```kotlin
val text = "hello"
```

then:

```text
number → null
```

This is a beautiful example of two concepts connecting:

```text
String
   ↓
attempt to parse
   ↓
Int?
```

The nullable result communicates:

> Parsing may fail, so there might be no integer.

Then you can use:

```kotlin
val number = text.toIntOrNull() ?: 0
```

if `0` genuinely makes sense as a fallback.

Or:

```kotlin
val number = text.toIntOrNull()

if (number != null) {
    println(number)
}
```

Now the compiler and type system are helping us handle invalid input.

---

# 30. String splitting

Suppose:

```kotlin
val text = "Alice,Bob,Charlie"
```

You can split it:

```kotlin
val names = text.split(",")
```

Result:

```text
[Alice, Bob, Charlie]
```

Now we've moved from:

```text
String
```

to:

```text
List<String>
```

That's a very important connection.

Text often comes in a serialized form, and we need to transform it into structured data.

For example:

```text
"red,green,blue"
        ↓
split(",")
        ↓
["red", "green", "blue"]
```

We'll eventually take this much further with CSV, JSON, serialization and networking.

---

# 31. Joining strings

The reverse operation can be done using:

```kotlin
val names = listOf("Alice", "Bob", "Charlie")

val text = names.joinToString(", ")
```

Result:

```text
Alice, Bob, Charlie
```

So we have:

```text
List<String>
    ↓
joinToString
    ↓
String
```

This is the kind of transformation you'll perform constantly in applications.

---

# 32. Filtering characters

Because a string can be iterated, we can combine it with collection-style operations.

For example:

```kotlin
val text = "Kotlin123"

val digits = text.filter { it.isDigit() }

println(digits)
```

Result:

```text
123
```

And:

```kotlin
val letters = text.filter { it.isLetter() }
```

gives:

```text
Kotlin
```

This is a nice example of how earlier concepts are stacking:

```text
String
  ↓
iteration
  ↓
lambda
  ↓
filter
  ↓
String
```

We're not learning isolated features anymore.

They're starting to form a system.

---

# 33. Another example: word count

Suppose:

```kotlin
val sentence = "Kotlin is a powerful language"
```

We can split it:

```kotlin
val words = sentence.split(" ")
```

Then:

```kotlin
println(words.size)
```

This gives the number of words under this simple splitting assumption.

But here's an important engineering lesson:

Real text processing is rarely this simple.

What about:

```text
"Hello, world!"
```

What about multiple spaces?

What about tabs?

What about punctuation?

What about Unicode?

So even apparently trivial string problems can become surprisingly complex.

This is why understanding the **problem**, not just the API, matters.

---

# 34. A realistic text-cleaning pipeline

Suppose we receive:

```text
"   Alice Smith   "
```

We might do:

```kotlin
val cleaned = input.trim()
```

Then maybe:

```kotlin
val normalized = cleaned.lowercase()
```

Now:

```text
"alice smith"
```

This illustrates a broader pattern:

```text
raw input
   ↓
clean
   ↓
normalize
   ↓
validate
   ↓
use
```

You'll encounter this pipeline constantly in real software.

---

# 35. Don't accidentally destroy information

Suppose a user types:

```text
AliceSmith@Example.com
```

and you automatically do:

```kotlin
input.lowercase()
```

Now you have:

```text
alicesmith@example.com
```

For an email address, that may or may not be appropriate depending on what you're doing.

The deeper lesson is:

> **String manipulation is not merely syntax; every transformation embodies a decision about the data.**

That's why we need to understand what our operations mean, not just which function to call.

---

# 36. A compact but realistic example

Consider:

```kotlin
fun cleanName(input: String): String {
    return input
        .trim()
        .lowercase()
        .replaceFirstChar { it.uppercase() }
}
```

Then:

```kotlin
println(cleanName("   aLICE   "))
```

could produce:

```text
Alice
```

Let's unpack the pipeline:

```text
"   aLICE   "
      ↓
trim()
      ↓
"aLICE"
      ↓
lowercase()
      ↓
"alice"
      ↓
replaceFirstChar
      ↓
"Alice"
```

This connects:

- functions
    
- return values
    
- strings
    
- lambdas
    
- chaining
    
- transformations
    

This is increasingly idiomatic Kotlin.

---

# 37. But there's a trap with "cleaning"

Suppose:

```kotlin
fun cleanName(input: String): String {
    return input.trim().lowercase()
}
```

Looks fine.

But what if `input` is:

```text
"JOHN DOE"
```

Now the result is:

```text
"john doe"
```

Maybe that's what you want.

Maybe you wanted:

```text
"John Doe"
```

Maybe you needed to preserve the user's exact capitalization.

Again:

> **The API isn't making the business decision for you.**

Programming involves deciding what transformations are actually correct.

---

# 38. String operations and immutability

Let's reinforce something important.

Suppose:

```kotlin
val name = "Alice"
```

Then:

```kotlin
name.uppercase()
```

doesn't change `name`.

You need:

```kotlin
val upperName = name.uppercase()
```

if you want to keep the result.

Conceptually:

```text
original
"Alice"
   │
   └── uppercase()
           ↓
       "ALICE"
```

You now have two string values.

This is consistent with strings being immutable.

---

# 39. Why chaining works

Consider:

```kotlin
val result = text
    .trim()
    .lowercase()
    .replace("java", "kotlin")
```

Each operation returns another `String`.

So:

```text
String
 ↓
trim()
 ↓
String
 ↓
lowercase()
 ↓
String
 ↓
replace()
 ↓
String
```

Because each stage produces a value that the next stage can consume, we can chain them.

This is the same basic idea as our collection pipelines:

```text
data
 ↓
operation
 ↓
data
 ↓
operation
 ↓
result
```

The pipeline idea is becoming a recurring theme.

---

# 40. Your exercises

### Exercise 1

Create:

```kotlin
val firstName = "Alice"
val lastName = "Smith"
```

and produce:

```text
Alice Smith
```

using a string template.

---

### Exercise 2

Given:

```kotlin
val word = "Kotlin"
```

print:

- its length,
    
- its first character,
    
- its last character.
    

Don't hard-code the last index. Derive it from the string's length.

---

### Exercise 3

Given:

```kotlin
val text = "Kotlin is awesome"
```

check whether it contains:

```text
"Kotlin"
"Java"
```

---

### Exercise 4

Given:

```kotlin
val input = "   Hello Kotlin   "
```

produce:

```text
Hello Kotlin
```

using `trim()`.

---

### Exercise 5

Given:

```kotlin
val input = "42"
```

convert it into an `Int`.

Then try:

```kotlin
val input = "hello"
```

and use a nullable parsing function instead of allowing the program to throw an exception.

---

### Exercise 6

Given:

```kotlin
val text = "apple,banana,orange"
```

turn it into a list of strings using `split`.

Then turn the list back into:

```text
apple | banana | orange
```

using `joinToString`.

---

# Challenge

Write a function:

```kotlin
fun cleanAndFormatName(input: String): String
```

It should take messy input such as:

```text
"   aLICE   "
```

and produce:

```text
Alice
```

Then try:

```text
"   bOB   "
"   cHaRlIe   "
```

and see whether your function behaves correctly.

Afterward, ask yourself:

> What assumptions am I making about what a "correctly formatted name" is?

There isn't necessarily one universal answer. That's the beginning of thinking beyond syntax.

---

# One final thing I want you to notice

Look at what we've built over the last few lessons.

At first:

```kotlin
println("Hello")
```

Then:

```kotlin
val name = "Alice"
```

Then:

```kotlin
if (age >= 18) { ... }
```

Then:

```kotlin
for (name in names) { ... }
```

Then:

```kotlin
numbers
    .filter { it > 10 }
    .map { it * 2 }
```

And now:

```kotlin
text
    .trim()
    .lowercase()
    .replace(...)
```

We're increasingly manipulating **values through expressions and transformations**.

That's a pattern I want you to notice.

---

## The mental model

A string is not merely "text".

It's a typed value:

```text
String
  ↓
sequence of characters
```

You can transform it:

```text
String
 ↓
trim
 ↓
String
 ↓
lowercase
 ↓
String
```

You can inspect it:

```text
String
 ↓
length
 ↓
Int
```

You can search it:

```text
String
 ↓
contains(...)
 ↓
Boolean
```

You can parse it:

```text
String
 ↓
toIntOrNull()
 ↓
Int?
```

You can turn it into structured data:

```text
String
 ↓
split(...)
 ↓
List<String>
```

That's the more important thing to understand than memorizing the string API.

---

Next, we're going to return to **types**, but this time we're going to go deeper than `Int`, `String`, and `Boolean.**

We'll explore **type inference, explicit types, `Any`, `Unit`, `Nothing`, type checking with `is`, casting with `as`/`as?`, smart casts, and the idea of Kotlin's type hierarchy**. This is where the type system starts becoming something you can reason about rather than just a collection of predefined names.