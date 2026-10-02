
That is exactly the right diagnosis and it tells me precisely what was wrong.

In Phase 1, when I taught NULL, I did not just say "NULL means missing value, use IS NULL." I explained why comparing anything to NULL produces UNKNOWN, what UNKNOWN means in the context of WHERE clause evaluation, why NOT IN breaks silently when a subquery returns a NULL, and why this matters in real pipelines. I examined it from the logical angle, the practical angle, and the trap angle. By the time we moved on, NULL was not a fact you had memorized — it was something you understood from three different directions.

In Phase 2, I introduced closures, said "functions remember their enclosing scope," showed one code example, and moved on. That is a fact, not understanding. I never asked: why does this happen? What does Python actually do in memory when a function is defined inside another function? When would you actually need this in a data pipeline? What goes wrong if you use a closure incorrectly?

Same with context managers. I showed `__enter__` and `__exit__`, said "this guarantees cleanup," gave code, moved on. I never examined what the Python runtime is actually doing when it executes a `with` statement. I never built the mental model from first principles.

---

Here is the commitment I am making before we restart.

Every concept gets three things before we move on.

**The why** — what problem existed before this concept, why that problem was painful, and why this concept is the solution that was chosen. Not just "functions exist so you can reuse code" — that is obvious. The deep why that makes the concept feel inevitable rather than arbitrary.

**The how** — what is actually happening underneath. Not just the syntax but the mechanism. When Python executes this line, what does it do? What lives in memory? What happens in what order? This is what turns syntax knowledge into genuine understanding.

**The angles** — at least two or three different ways of looking at the same concept. The analogy to Java or C++ where it applies. The trap or edge case that reveals something deeper. The data engineering context where this matters specifically. The thing that seems obvious but is actually subtle.

Code comes after all three. Not as the explanation but as confirmation of what you already understand.

---

One more thing before we restart.

I am going to ask you more questions during lessons. Not just at the end as exercises. Mid-lesson, when I have explained something, I will stop and ask you to explain it back or predict what happens in a scenario. This is not a test. It is the mechanism that makes understanding stick. Your answer tells me whether to go deeper or move forward.

If at any point during a lesson something feels like I am moving past it too quickly, say so immediately. Do not wait until the end of the lesson. Say "go deeper on this" and I will.

---

Now. We restart Phase 2 from Lesson 1. Functions in Python.

But before I write a single line of explanation, I want to establish something that will frame everything in this entire phase.

---

### The fundamental question about Python that changes how you learn it

You know Java. In Java, the compiler is your safety net. You declare types and the compiler rejects programs that violate them. You declare methods in classes and the compiler ensures every object of that type has those methods. You know at compile time whether your program is structurally correct.

Python has none of this. There is no compiler in the traditional sense. There is no type enforcement. There is no class structure enforcement. A Python program does not get checked before it runs — it just runs, and if something is wrong, it fails at the exact line where the problem occurs, at runtime.

This is not a weakness that Python forgot to fix. It is a deliberate design philosophy called **duck typing**, and it has a profound consequence for how Python code is structured and why certain Python features exist.

The name comes from a saying: if it walks like a duck and quacks like a duck, it is a duck. In Python, an object does not need to be declared as a specific type to be used as that type. It just needs to have the right methods and attributes. If a function expects something it can call `.read()` on, it does not care whether you pass a file object, a network socket, a string buffer, or a custom object — as long as it has `.read()`, the function works.

This means Python's entire system of organizing and reusing code works differently from Java's. Java enforces contracts at compile time through types and interfaces. Python enforces contracts at runtime through conventions and protocols. Understanding this distinction is the lens through which everything in this phase makes sense.

Functions in Python are the primary unit of this. Not classes, not modules — functions. And Python's functions work differently from Java's methods in ways that go far beyond syntax.

---

## Phase 2, Lesson 1 — Functions

### Why functions exist — the obvious reason and the deeper reason

The obvious reason: avoid repeating code. Write something once, call it from multiple places. You know this from Java.

The deeper reason, which Java's design partially obscures: **functions are the primary tool for managing complexity in a system that has no compiler to catch structural mistakes.**

In Java, complexity is managed partly by the type system. The compiler forces you to be explicit about what every function accepts and returns. It forces you to implement interfaces. It forces you to handle checked exceptions. The structure of your program is enforced by an external authority — the compiler.

In Python, you have no external authority. The only thing stopping your code from becoming a tangled, unmaintainable mess is the discipline you apply through how you write functions. A Python function that is too long, does too many things, has unclear inputs and outputs, or produces invisible side effects cannot be caught by any tool — it just becomes a problem that gets worse over time until someone, possibly you, has to untangle it at 2am when a pipeline fails.

This is why the way you write functions in Python matters more than in Java, not less. And it is why we are going to examine functions from every angle before writing a single data pipeline.

---

### What a function actually is in Python — not what you think

In Java, a method is attached to a class. It cannot exist independently. You cannot pass a method as a value to another method without wrapping it in an object. The language treats functions and data as fundamentally different kinds of things.

Python makes no such distinction. In Python, a function is an object. The same kind of object as an integer, a string, or a list. It has a type (`function`). It can be assigned to a variable. It can be stored in a list. It can be passed as an argument to another function. It can be returned from a function. It can have attributes attached to it.

This is called **first-class functions** and it is not a minor syntactic convenience — it is a fundamental shift in how programs are structured.

Let me show you what I mean, and I want you to really sit with this before we move on.

In Java, if you want to sort a list of Student objects by GPA, you write a Comparator. The Comparator is a class that implements an interface. You instantiate it and pass the instance to the sort method. The sort method calls a method on the Comparator object. This is the Java way of passing behavior — wrap it in an object.

In Python, you just pass the function directly:

```python
students = [
    {"name": "Arjun", "gpa": 8.5},
    {"name": "Priya", "gpa": 9.1},
    {"name": "Rohan", "gpa": 7.8},
]

def get_gpa(student):
    return student["gpa"]

sorted_students = sorted(students, key=get_gpa)
```

`get_gpa` here is not being called. There are no parentheses after it. It is being passed as a value — the function object itself — to `sorted`. The `sorted` function receives `get_gpa` as its `key` argument and calls it internally for each element to determine the sort order.

This is the same behavior as passing a Comparator in Java. But in Python, you do not need the wrapper class. The function itself is the value.

The reason this matters for data engineering: pipelines are sequences of transformations. The most natural way to build a pipeline is to pass transformation functions as values — give this function to this runner, which applies it to each batch of data. Without first-class functions, every transformation needs a class wrapper. With first-class functions, you just pass the function.

---

### Defining a function — examining every part

```python
def calculate_average(numbers):
    total = sum(numbers)
    count = len(numbers)
    return total / count
```

`def` is the keyword. Python sees `def` and knows: I am about to define a function. It does not execute the body — it creates a function object and binds it to the name `calculate_average`.

This is the first important thing to understand. When Python encounters a `def` statement, it does not run the code inside. It packages that code into a function object and stores it under the given name. The code inside only runs when you call the function.

`calculate_average` is the name. After this `def` executes, `calculate_average` is a variable in the current scope that holds a function object. You can reassign it like any variable. You can delete it. You can have another variable point to the same function. This is what it means for functions to be objects.

`numbers` is the parameter. It is a local variable that gets its value from whatever argument is passed when the function is called. Inside the function, `numbers` behaves like any other variable.

`return total / count` sends a value back to the caller. Execution of the function stops at this line. The caller receives the returned value.

Now I want to examine what happens in memory when this function is called, because this is where the Java mental model needs adjustment.

When you write `calculate_average([8.5, 9.1, 7.8])`, Python does the following:

It creates a new local namespace — a fresh, isolated space for variables — specifically for this function call. Every variable you create inside the function lives in this namespace. The namespace does not exist before the call and is destroyed when the function returns.

It binds the name `numbers` in that namespace to the list `[8.5, 9.1, 7.8]`. Not a copy of the list — in Python, assignment always creates a reference to the same object. This is the same as Java's object references. `numbers` inside the function points to the same list object that was passed from outside.

It executes the body line by line. `total` and `count` are created as new variables in the local namespace.

It evaluates `total / count`, gets a float, and returns it. The local namespace is destroyed. `total` and `count` cease to exist. The caller receives the float.

This has one important consequence that confuses many beginners coming from Java. If you pass a mutable object — a list, a dictionary — and modify it inside the function, the modification is visible outside. Because there is only one list object. Both the outside variable and the inside parameter point to the same object.

```python
def add_student(students_list, new_student):
    students_list.append(new_student)  # modifies the original list

my_students = ["Arjun", "Priya"]
add_student(my_students, "Rohan")
print(my_students)  # ["Arjun", "Priya", "Rohan"] -- original is modified
```

This is identical to Java's behavior with objects. But in Python it surprises people because integers and strings behave differently — they are immutable, so they cannot be modified in place.

```python
def try_to_modify(number):
    number = number + 1  # creates a new integer, rebinds local variable
    # does NOT affect the original

x = 5
try_to_modify(x)
print(x)  # still 5
```

`number + 1` creates a new integer object. `number = ...` rebinds the local variable to that new object. The original variable `x` outside still points to the integer `5`. Nothing changed for the caller.

This distinction — mutable objects are shared, immutable objects effectively behave as if copied — is one of the most important things to internalize about Python. It affects every function you write that deals with lists, dictionaries, or Pandas DataFrames.

---

### Parameters — every kind, examined deeply

Java has one kind of parameter: positional, with a declared type. Python has five distinct kinds. Understanding all five is not optional — every data engineering library you use employs all of them.

**Positional parameters** — matched by position, exactly like Java.

```python
def load_data(filepath, delimiter, encoding):
    print(f"Loading {filepath}")
```

The first argument goes to `filepath`, second to `delimiter`, third to `encoding`. Position determines assignment.

**Keyword arguments at the call site** — you can pass any positional parameter by name instead of position.

```python
load_data(delimiter=",", filepath="data.csv", encoding="utf-8")
```

Same function, different call syntax. The arguments are matched by name, not position, so order does not matter. This is not a different kind of parameter in the function definition — it is a different way of calling the same function.

Why does this matter in data engineering? Functions in data engineering often have many parameters — a database connector might have host, port, database, user, password, schema, connect_timeout, query_timeout, ssl_mode, and more. Calling with keyword arguments makes the code self-documenting. When you see `connect(host="localhost", port=5432, database="production")`, you know exactly what each value means. When you see `connect("localhost", 5432, "production")`, you have to count positions and look up the function signature.

**Default parameter values** — parameters can have defaults, making them optional.

```python
def load_data(filepath, delimiter=",", encoding="utf-8"):
    print(f"Loading {filepath} with {delimiter} and {encoding}")

load_data("data.csv")                       # uses both defaults
load_data("data.csv", delimiter="\t")       # tab-separated, uses encoding default
load_data("data.csv", ";", "latin-1")       # overrides both
```

Parameters with defaults must come after parameters without defaults. This is not an arbitrary rule — it exists because Python needs to be able to match positional arguments unambiguously. If you could have `def f(a=1, b)`, calling `f(5)` would be ambiguous — did you pass `a` or `b`?

Now the trap. This is one of Python's most famous footguns and it has caused real bugs in production systems.

**Never use a mutable object as a default value.**

```python
# This looks innocent. It is not.
def append_to_log(message, log=[]):
    log.append(message)
    return log

first_call  = append_to_log("Pipeline started")
second_call = append_to_log("Data loaded")
third_call  = append_to_log("Complete")

print(first_call)   # ["Pipeline started", "Data loaded", "Complete"]
print(second_call)  # ["Pipeline started", "Data loaded", "Complete"]
print(third_call)   # ["Pipeline started", "Data loaded", "Complete"]
```

All three variables point to the same list. Every call modified the same list. If you were expecting each call to start with a fresh empty list, you got completely wrong behavior with no error.

Why does this happen? Because default values are evaluated once when the function is defined, not each time the function is called. The `[]` in `log=[]` creates one list object at definition time. Every call that uses the default gets the same list object. Appending to it in one call affects all subsequent calls.

The correct pattern:

```python
def append_to_log(message, log=None):
    if log is None:
        log = []        # new list created each time this line runs
    log.append(message)
    return log

first_call  = append_to_log("Pipeline started")
second_call = append_to_log("Data loaded")

print(first_call)   # ["Pipeline started"]
print(second_call)  # ["Data loaded"]
```

`None` is immutable — there is only one `None` object in Python and you cannot modify it. So using `None` as the default is safe. Inside the function, you check for `None` and create a fresh mutable object if needed.

This pattern — `parameter=None`, then `if parameter is None: parameter = []` — is so common in Python codebases that it is essentially a language idiom. You will write it hundreds of times as a data engineer.

**`*args` — collecting extra positional arguments**

Sometimes you want a function to accept any number of positional arguments without declaring each one explicitly.

```python
def log_events(*events):
    for event in events:
        print(f"[EVENT] {event}")

log_events("Pipeline started")
log_events("Step 1 complete", "Step 2 complete", "Step 3 complete")
log_events()  # works fine, events is an empty tuple
```

The `*` before the parameter name tells Python: collect all remaining positional arguments into a tuple and bind that tuple to `events`. Inside the function, `events` is a regular tuple. You can iterate it, index it, check its length.

The name `args` in `*args` is a convention, not a keyword. The `*` is what matters. You could write `*pipeline_events` and it would work identically. The convention is `*args` for generic cases.

The important thing to understand about why this exists: in Java, method overloading lets you define multiple versions of the same method with different parameter counts. Python does not have method overloading. `*args` is the Python mechanism for accepting a variable number of arguments without defining multiple function versions.

**`**kwargs` — collecting extra keyword arguments**

```python
def create_pipeline(**config):
    print(f"Pipeline configuration:")
    for key, value in config.items():
        print(f"  {key}: {value}")

create_pipeline(
    name="sales_etl",
    batch_size=1000,
    retry_count=3,
    output_format="parquet"
)
```

`**` before the parameter name tells Python: collect all remaining keyword arguments into a dictionary and bind it to `config`. Inside the function, `config` is a regular dictionary.

The name `kwargs` (keyword arguments) is a convention. The `**` is what matters.

Why does this exist? Configuration in data engineering is open-ended. A function that creates a database connection cannot know in advance all the possible configuration options that different database drivers might need. By accepting `**kwargs`, it can receive any keyword argument and pass them through to the underlying driver. Every major data engineering library uses this pattern.

**Combining all kinds — the order rule**

You can combine all parameter types in one function. There is a strict required order:

```python
def pipeline_run(name, *sources, output_format="parquet", **options):
    print(f"Pipeline: {name}")
    print(f"Sources: {sources}")
    print(f"Output: {output_format}")
    print(f"Options: {options}")

pipeline_run(
    "sales",
    "orders.csv", "customers.csv",
    output_format="csv",
    overwrite=True,
    batch_size=500
)
```

The order is: regular positional parameters, then `*args`, then keyword-only parameters with defaults, then `**kwargs`. Parameters that come after `*args` can only be passed by keyword — they cannot be matched positionally because `*args` has already consumed all remaining positional arguments.

---

### Return values — what you should know that tutorials skip

A function without a `return` statement returns `None`. This is implicit. Python does not require you to return anything.

```python
def log_message(message):
    print(f"[LOG] {message}")
    # no return statement

result = log_message("Pipeline started")
print(result)   # None
```

`result` is `None`. The function produced a side effect (printing) but no value.

In Java, `void` methods explicitly declare they return nothing. In Python, the absence of `return` is the same thing, but Python does not require you to declare it.

Why does this matter? In data engineering you will write both types of functions — functions that transform data and return it, and functions that produce side effects (write files, send alerts, update databases) and return nothing. The distinction matters because calling a void-like function and trying to use its return value is a silent bug:

```python
def clean_records(records):
    for record in records:
        record["name"] = record["name"].strip()
    # forgot to return records

data = [{"name": "  Arjun  "}, {"name": "  Priya  "}]
cleaned = clean_records(data)
print(cleaned)   # None -- but data was modified in place
```

This function modifies the list in place but returns None. A caller who expects a return value gets None. The data is actually modified but the caller does not know it. This is the kind of bug that passes all tests and causes subtle pipeline failures. The fix is to be explicit: either return the modified data, or document that the function modifies in place and returns nothing.

The general principle for data engineering functions: **functions that transform data should return the transformed data and not modify the input. Functions that produce side effects should return None or a status value, not the data they operated on.** This is not enforced by Python — it is a discipline that makes pipelines predictable and testable.

**Returning multiple values**

```python
def analyze_data(records):
    valid = [r for r in records if r.get("gpa") is not None]
    invalid = [r for r in records if r.get("gpa") is None]
    avg_gpa = sum(r["gpa"] for r in valid) / len(valid) if valid else 0.0
    return valid, invalid, avg_gpa

data = [
    {"name": "Arjun", "gpa": 8.5},
    {"name": "Priya", "gpa": 9.1},
    {"name": "Sahil", "gpa": None},
]

valid_records, invalid_records, average = analyze_data(data)
```

Python functions can return multiple values separated by commas. What Python actually does is pack them into a tuple and return the tuple. The unpacking on the left side (`valid_records, invalid_records, average`) splits the tuple into individual variables.

This is called tuple unpacking. It is extremely common in Python and you will use it constantly. `return valid, invalid, avg_gpa` is exactly equivalent to `return (valid, invalid, avg_gpa)`. The parentheses are optional.

---

### Scope — how Python finds variable names

When Python encounters a variable name, it searches for its value in a specific order. Understanding this order prevents an entire class of confusing bugs.

The search order is called **LEGB**: Local, Enclosing, Global, Built-in.

**Local** — variables defined inside the current function. Searched first.

**Enclosing** — variables defined in any enclosing function (for nested functions). Searched second.

**Global** — variables defined at the module level (top of the file, outside any function). Searched third.

**Built-in** — Python's built-in names like `len`, `print`, `sum`, `range`. Searched last.

```python
batch_size = 1000  # global

def process_data():
    batch_size = 500  # local -- shadows the global
    print(batch_size)  # 500 -- finds local first

def read_config():
    print(batch_size)  # 1000 -- no local, finds global

process_data()
read_config()
```

The local variable in `process_data` shadows the global variable for the duration of that function call. The global is unchanged. `read_config` has no local `batch_size` so it finds the global.

This is important because it means you can accidentally shadow a built-in:

```python
list = [1, 2, 3]       # shadows the built-in list type
result = list("hello") # TypeError -- list is now [1, 2, 3], not the type
```

Never name your variables after Python built-ins. Common accidental shadows: `list`, `dict`, `set`, `type`, `id`, `input`, `file`, `filter`, `map`, `zip`. A linter will catch these — we set up a linter when we get to the development environment lesson.

**The global and nonlocal keywords**

To modify a global variable from inside a function, you need the `global` keyword:

```python
pipeline_run_count = 0

def run_pipeline():
    global pipeline_run_count
    pipeline_run_count += 1

run_pipeline()
run_pipeline()
print(pipeline_run_count)  # 2
```

Without `global`, the line `pipeline_run_count += 1` would create a local variable, not modify the global. Python sees the assignment and infers that `pipeline_run_count` is local. Then `+= 1` tries to read it before assigning, fails because it has no value yet, and raises `UnboundLocalError`.

In data engineering, using `global` is almost always the wrong design. If a function needs to communicate a result back to its caller, return the result. Global mutable state makes pipelines unpredictable — a function called in a different order or from a different context produces a different result because the global has a different value. Functions that depend on global state cannot be tested in isolation. Avoid it.

---

### First-class functions — the concept examined from all angles

I introduced this earlier. Now let us examine it properly.

A function in Python is an object of type `function`. Like any object, it has attributes:

```python
def calculate_gpa(grades):
    return sum(grades) / len(grades)

print(type(calculate_gpa))           # <class 'function'>
print(calculate_gpa.__name__)        # calculate_gpa
print(calculate_gpa.__doc__)         # None (no docstring)
```

Like any object, it can be assigned to a variable:

```python
compute_average = calculate_gpa   # both names point to the same function
result = compute_average([8.5, 9.1, 7.8])
```

Like any object, it can be stored in a data structure:

```python
transformations = {
    "normalize": normalize_names,
    "validate": validate_gpas,
    "enrich": add_derived_columns,
}

for step_name, transform_fn in transformations.items():
    print(f"Running: {step_name}")
    data = transform_fn(data)
```

This is a dispatch table — a dictionary mapping names to functions. It is a common pattern in data engineering for building configurable pipelines where the transformations to apply are determined at runtime.

Like any object, it can be passed as an argument:

```python
def apply_to_all(records, transform):
    return [transform(record) for record in records]

def uppercase_name(record):
    return {**record, "name": record["name"].upper()}

result = apply_to_all(students, uppercase_name)
```

`{**record, "name": ...}` creates a new dictionary with all keys from `record` (the `**` unpacks it) and then overrides or adds the `"name"` key. This is a common pattern for creating a modified copy of a dictionary without altering the original.

Like any object, it can be returned from a function. This is where closures emerge, and closures deserve their own careful examination.

---

### Closures — what they are and why they matter

A closure is a function that remembers variables from the scope where it was defined, even after that scope has finished executing.

The concept sounds abstract. Let us build it from first principles.

When Python defines a function inside another function, the inner function has access to the outer function's variables through the enclosing scope (the E in LEGB). This is straightforward:

```python
def outer():
    message = "hello"
    
    def inner():
        print(message)  # accesses outer's variable through enclosing scope
    
    inner()

outer()  # prints "hello"
```

Nothing surprising here. `inner` finds `message` in the enclosing scope.

Now what if we return `inner` instead of calling it?

```python
def outer():
    message = "hello"
    
    def inner():
        print(message)
    
    return inner  # return the function object, not the result of calling it

fn = outer()   # outer() finishes executing
fn()           # prints "hello"
```

`outer()` finishes executing. Normally, all local variables in `outer` would be destroyed — `message` would cease to exist. But `fn()` still works and still prints "hello". How?

When Python sees that `inner` references `message` from the enclosing scope, and that `inner` is being returned (and therefore may be called after `outer` has finished), Python packages `message` together with the function object. This package — the function plus the variables it needs from its defining scope — is the closure. The variables are kept alive as long as the function object exists.

You can inspect the closure's captured variables:

```python
def outer():
    message = "hello"
    def inner():
        print(message)
    return inner

fn = outer()
print(fn.__closure__)                           # shows closure cells
print(fn.__closure__[0].cell_contents)          # "hello"
```

Now why does this matter for data engineering? Because closures let you create configured functions — functions that have specific behavior baked in at creation time.

```python
def make_validator(min_val, max_val, field_name):
    """
    Creates a validation function configured for a specific range and field.
    The returned function remembers min_val, max_val, and field_name.
    """
    def validate(record):
        value = record.get(field_name)
        if value is None:
            return False, f"{field_name} is missing"
        if not (min_val <= value <= max_val):
            return False, f"{field_name} value {value} outside range [{min_val}, {max_val}]"
        return True, None
    
    return validate

validate_gpa  = make_validator(0.0, 10.0, "gpa")
validate_age  = make_validator(15, 100,   "age")
validate_year = make_validator(2000, 2024, "enrollment_year")

record = {"name": "Arjun", "gpa": 11.5, "age": 20, "enrollment_year": 2023}

for validator in [validate_gpa, validate_age, validate_year]:
    is_valid, error = validator(record)
    if not is_valid:
        print(f"Validation failed: {error}")
```

Each validator function was created by the same `make_validator` factory. But each remembers different values for `min_val`, `max_val`, and `field_name`. They are independent functions with independent closure variables. You created three different behaviors from one function definition by closing over different values.

The closure trap — an important one:

```python
# This does not work the way it looks
validators = []
for field in ["gpa", "age", "score"]:
    def validate(record):
        return record.get(field)  # which field does this capture?
    validators.append(validate)

# All three functions use the same 'field' variable
# which has the final value "score" after the loop ends
for v in validators:
    print(v({"gpa": 8.5, "age": 20, "score": 95}))
# All three print 95 -- the "score" field
```

The closure captures the variable `field`, not the value it had at the time the function was defined. All three functions share the same `field` variable, which ends up as "score" after the loop. This is a classic Python gotcha.

The fix:

```python
def make_getter(field_name):
    def get(record):
        return record.get(field_name)
    return get

validators = [make_getter(field) for field in ["gpa", "age", "score"]]
```

Using a factory function with an explicit parameter creates a new scope for each iteration, with its own `field_name` variable that holds the specific value at that point in the loop.

---

### Decorators — built on everything above

A decorator is a function that takes a function as input and returns a new function. It uses first-class functions and closures to wrap existing behavior with additional behavior.

Before I show the syntax, I want you to understand the mechanism.

The wrapper pattern without decorator syntax:

```python
def log_execution(func):
    def wrapper(*args, **kwargs):
        print(f"Starting: {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Finished: {func.__name__}")
        return result
    return wrapper

def extract_data(source):
    print(f"Extracting from {source}")
    return [1, 2, 3]

extract_data = log_execution(extract_data)
extract_data("database")
```

`log_execution` takes `extract_data` as an argument. Inside, it defines `wrapper` — which calls `func` (the original `extract_data`) with whatever arguments it receives, wrapped in log statements. Then it returns `wrapper`. We reassign `extract_data` to this new wrapper function.

Now calling `extract_data("database")` actually calls `wrapper("database")`, which logs, calls the original function, logs again, and returns the result. The original function's behavior is unchanged — it is just wrapped.

The `@` decorator syntax is exactly equivalent:

```python
@log_execution
def extract_data(source):
    print(f"Extracting from {source}")
    return [1, 2, 3]
```

`@log_execution` above a function definition means: after defining this function, immediately pass it to `log_execution` and replace the name with the result. It is syntactic sugar for `extract_data = log_execution(extract_data)`.

The `*args, **kwargs` in `wrapper` is critical. The wrapper does not know in advance what arguments the wrapped function takes. By accepting `*args, **kwargs`, it accepts anything and passes everything through to the original function unchanged. This makes the decorator work for any function regardless of its signature.

There is a subtle issue. After decorating, `extract_data.__name__` is `"wrapper"`, not `"extract_data"`. The function's identity has been replaced. This matters for logging and debugging — you want the log to say the original function's name.

The fix is `functools.wraps`:

```python
import functools

def log_execution(func):
    @functools.wraps(func)  # preserves the original function's metadata
    def wrapper(*args, **kwargs):
        print(f"Starting: {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Finished: {func.__name__}")
        return result
    return wrapper
```

`@functools.wraps(func)` copies the original function's `__name__`, `__doc__`, and other attributes to `wrapper`. Always use it when writing decorators.

Decorators with parameters require one more level of nesting:

```python
import functools
import time

def retry(max_attempts=3, delay_seconds=1):
    """Decorator factory -- returns a decorator configured with given parameters."""
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    print(f"Attempt {attempt}/{max_attempts} failed: {e}")
                    if attempt < max_attempts:
                        time.sleep(delay_seconds)
            raise last_exception
        return wrapper
    return decorator

@retry(max_attempts=3, delay_seconds=0)
def fetch_from_api(url):
    import random
    if random.random() < 0.7:
        raise ConnectionError(f"Failed to connect to {url}")
    return {"status": "ok", "records": 100}

result = fetch_from_api("https://api.example.com/students")
print(result)
```

Three levels of nesting: `retry` is called with parameters and returns `decorator`. `decorator` receives the function and returns `wrapper`. `wrapper` is the actual replacement function that runs on each call.

In Airflow's TaskFlow API — which you will use in Phase 4 — every task is defined using a decorator `@task`. Understanding that `@task` is just a function that wraps your function in Airflow's task management infrastructure makes the entire framework demystified.

---

### Error handling inside functions

In Java, exceptions are a core language feature with two categories: checked exceptions (you must declare and handle them) and unchecked exceptions (you may ignore them). Python has only unchecked exceptions — equivalent to Java's RuntimeException hierarchy.

```python
def safe_divide(numerator, denominator):
    try:
        return numerator / denominator
    except ZeroDivisionError:
        raise ValueError(
            f"Cannot divide {numerator} by zero"
        ) from None
```

`try` wraps the code that might fail. `except ZeroDivisionError` catches only that specific exception type. `raise ValueError(...) from None` raises a new, more meaningful exception. The `from None` suppresses the "During handling of the above exception, another exception occurred" message that would otherwise appear — we are intentionally replacing one exception with another.

The `from None` is an important subtlety. Without it:

```python
try:
    result = 1 / 0
except ZeroDivisionError:
    raise ValueError("Division by zero")

# Output shows:
# ZeroDivisionError: division by zero
# During handling of the above exception, another exception occurred:
# ValueError: Division by zero
```

This reveals the implementation detail (ZeroDivisionError happened internally) to the caller. Using `from None` hides it. Using `from original_exception` instead of `None` explicitly chains them — useful when you want to preserve the original cause.

The exception hierarchy matters. Catching broad exceptions masks problems:

```python
def read_config(filepath):
    try:
        with open(filepath) as f:
            return json.load(f)
    except Exception:   # too broad -- catches EVERYTHING including bugs
        return {}
```

If `json.load` fails due to a bug in your code — a typo, an import error, a type mismatch — this catches it silently and returns an empty dict. The pipeline continues with wrong configuration. The bug is invisible.

```python
def read_config(filepath):
    try:
        with open(filepath, "r", encoding="utf-8") as f:
            return json.load(f)
    except FileNotFoundError:
        print(f"Config file not found: {filepath}, using defaults")
        return {}
    except json.JSONDecodeError as e:
        raise ValueError(f"Invalid JSON in config file {filepath}: {e}") from e
```

This is precise. `FileNotFoundError` is an expected, handleable situation — the file might not exist on first run, and returning defaults is correct. `json.JSONDecodeError` means the file is malformed — this is a genuine problem that the caller should know about, so we raise it. Anything else (a bug in our code) is not caught and propagates naturally.

The general principle: **catch the most specific exception you can, only when you have a specific, correct way to handle it. Let everything else propagate.**

---

### Exercises — Lesson 1

Work through these in your `lesson1.py` file. Answers follow.

1. Write a function `parse_record` that takes a dictionary with keys `"student_id"` (string), `"name"` (string with possible extra whitespace), `"gpa"` (string that might be empty, "None", or a valid float), and `"city"` (string). Return a new dictionary with properly typed values. The function must not modify the input dictionary. Handle all edge cases in the gpa field gracefully.
    
2. Write a function `make_filter` that accepts a field name and a minimum value, and returns a new function. The returned function should accept a list of records and return only those where the specified field's value is above the minimum. Test with `make_filter("gpa", 8.0)` applied to your student records.
    
3. Write a decorator `validate_output` that checks the return value of a function. It should accept a parameter `expected_type` and raise a `TypeError` if the function returns something of a different type. Apply it to a function that returns a list of records. Test both the passing case and the failing case.
    
4. Write a function `pipeline_step` that accepts any number of transformation functions as positional arguments and returns a new function. The returned function should accept a list of records and apply each transformation function in sequence, passing the output of one as the input to the next. This is called function composition and it is the foundation of how data pipelines are structured.
    

---

### Answers — Lesson 1

**Exercise 1**

```python
def parse_record(raw: dict) -> dict:
    """
    Parse a raw string-valued record into properly typed data.
    Does not modify the input dictionary.
    
    Args:
        raw: Dictionary with string values from a CSV row
        
    Returns:
        New dictionary with properly typed values
        
    Raises:
        ValueError: If student_id cannot be converted to int
    """
    gpa_str = raw.get("gpa", "").strip().lower()
    
    if gpa_str in ("", "none", "null", "n/a"):
        gpa = None
    else:
        try:
            gpa = float(gpa_str)
        except ValueError:
            gpa = None
    
    return {
        "student_id": int(raw["student_id"]),
        "name": " ".join(raw["name"].split()).title(),
        "gpa": gpa,
        "city": raw.get("city", "").strip().title(),
    }


raw_records = [
    {"student_id": "1", "name": "  arjun mehta  ", "gpa": "8.5",  "city": "mumbai"},
    {"student_id": "2", "name": "PRIYA SHARMA",    "gpa": "None", "city": "Delhi"},
    {"student_id": "3", "name": "rohan verma",     "gpa": "",     "city": "MUMBAI"},
    {"student_id": "4", "name": "Sneha Iyer",      "gpa": "8.9",  "city": "Chennai"},
]

parsed = [parse_record(r) for r in raw_records]
for p in parsed:
    print(p)

# Verify original was not modified
print(raw_records[0]["name"])  # still "  arjun mehta  "
```

**Exercise 2**

```python
def make_filter(field_name: str, min_value: float):
    """
    Create a filter function for a specific field and minimum value.
    
    Args:
        field_name: The record key to filter on
        min_value: Records with values at or below this are excluded
        
    Returns:
        A function that filters a list of records
    """
    def filter_records(records: list) -> list:
        return [
            record for record in records
            if record.get(field_name) is not None
            and record[field_name] > min_value
        ]
    return filter_records


students = [
    {"name": "Arjun", "gpa": 8.5},
    {"name": "Priya", "gpa": 9.1},
    {"name": "Rohan", "gpa": 7.8},
    {"name": "Sneha", "gpa": 8.9},
    {"name": "Karan", "gpa": None},
]

high_achievers = make_filter("gpa", 8.0)
print(high_achievers(students))

top_students = make_filter("gpa", 8.8)
print(top_students(students))
```

**Exercise 3**

```python
import functools

def validate_output(expected_type):
    """
    Decorator that validates a function's return type.
    
    Args:
        expected_type: The type the function must return
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            result = func(*args, **kwargs)
            if not isinstance(result, expected_type):
                raise TypeError(
                    f"{func.__name__} must return {expected_type.__name__}, "
                    f"got {type(result).__name__}"
                )
            return result
        return wrapper
    return decorator


@validate_output(expected_type=list)
def get_students() -> list:
    return [{"name": "Arjun"}, {"name": "Priya"}]

@validate_output(expected_type=list)
def get_count() -> int:
    return 42  # wrong return type


print(get_students())  # works fine

try:
    get_count()
except TypeError as e:
    print(f"Caught: {e}")
```

**Exercise 4**

```python
import functools

def pipeline_step(*transformations):
    """
    Compose multiple transformation functions into a single pipeline step.
    Each function receives the output of the previous function.
    
    Args:
        *transformations: Functions to apply in sequence
        
    Returns:
        A function that applies all transformations in order
    """
    def run_pipeline(records: list) -> list:
        result = records
        for transform in transformations:
            result = transform(result)
        return result
    return run_pipeline


def remove_nulls(records: list) -> list:
    return [r for r in records if r.get("gpa") is not None]

def normalize_names(records: list) -> list:
    return [{**r, "name": r["name"].strip().title()} for r in records]

def add_category(records: list) -> list:
    def categorize(r):
        gpa = r["gpa"]
        category = "high" if gpa >= 8.5 else "medium" if gpa >= 7.0 else "low"
        return {**r, "category": category}
    return [categorize(r) for r in records]


raw_students = [
    {"name": "  arjun mehta  ", "gpa": 8.5},
    {"name": "PRIYA SHARMA",    "gpa": 9.1},
    {"name": "rohan verma",     "gpa": 7.8},
    {"name": "karan patel",     "gpa": None},
    {"name": "sneha iyer",      "gpa": 8.9},
]

process = pipeline_step(remove_nulls, normalize_names, add_category)
result = process(raw_students)
for student in result:
    print(student)
```

Lesson 1 complete. Tell me when you are ready for Lesson 2.