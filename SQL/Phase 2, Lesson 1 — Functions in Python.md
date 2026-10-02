

## Phase 2, Lesson 1 — Functions in Python

---

### Why functions matter more in data engineering than in most other fields

In data engineering you build pipelines — sequences of operations that transform data from one state to another. A pipeline that is written as one long script, top to bottom, with no functions, is a pipeline that cannot be tested, cannot be reused, cannot be debugged efficiently, and cannot be maintained by anyone including yourself six months later.

Every serious data engineering codebase is built from functions. Small, focused, named functions that do one thing each and compose together into larger operations. Learning to write functions well in Python is not a preliminary step before the real work — it is the real work.

---

### Defining a function — the syntax and what each part means

Create a new file in VS Code inside your `de_learning` folder. Call it `lesson1.py`.

```python
def greet(name):
    message = f"Hello, {name}"
    return message

result = greet("Arjun")
print(result)
```

`def` is the keyword that starts a function definition. `greet` is the function name. `name` inside the parentheses is a parameter — a variable that receives a value when the function is called. The indented block is the function body. `return` sends a value back to the caller.

Run this in your terminal:

```
cd de_learning
python lesson1.py
```

You should see `Hello, Arjun`.

This is structurally identical to a method in Java. The differences are no access modifier, no return type declaration, no parameter type declaration. Python does not enforce types at the syntax level — a function that expects a string will accept an integer without complaint, and only fail when the code inside tries to do something string-specific with it. We will deal with this properly when we cover type hints shortly.

---

### Parameters — the full picture

Python functions have several kinds of parameters. Understanding all of them is essential because you will encounter every kind in real data engineering libraries.

**Positional parameters** — the basic kind. Arguments are matched to parameters by position.

```python
def load_data(filepath, delimiter, encoding):
    print(f"Loading {filepath} with delimiter '{delimiter}' and encoding {encoding}")

load_data("data.csv", ",", "utf-8")
```

The first argument goes to filepath, second to delimiter, third to encoding. Order matters.

**Keyword arguments** — you can pass arguments by name, in any order.

```python
load_data(delimiter=",", filepath="data.csv", encoding="utf-8")
```

Same function, arguments passed by name. Order no longer matters. This is common in data engineering because functions often have many parameters and passing by name makes the code self-documenting — you can see exactly what each argument means without looking up the function definition.

**Default parameter values** — parameters can have defaults, making them optional.

```python
def load_data(filepath, delimiter=",", encoding="utf-8"):
    print(f"Loading {filepath} with delimiter '{delimiter}' and encoding {encoding}")

load_data("data.csv")                    # uses both defaults
load_data("data.csv", delimiter="\t")   # overrides delimiter only
load_data("data.csv", ";", "latin-1")   # overrides both
```

Parameters with defaults must come after parameters without defaults. This is enforced by Python — `def load_data(delimiter=",", filepath)` would raise a SyntaxError.

In data engineering, default parameters appear everywhere. A function that reads a file almost always defaults to UTF-8 encoding and comma delimiter. You override only when the data is different. This makes function calls concise in the common case while remaining flexible for edge cases.

**One critical rule about default values:** never use a mutable object (list, dictionary) as a default value. This is one of Python's most famous traps.

```python
# WRONG -- this default list is created once and shared across all calls
def append_record(record, records=[]):
    records.append(record)
    return records

# CORRECT -- create a new list each time the function is called
def append_record(record, records=None):
    if records is None:
        records = []
    records.append(record)
    return records
```

In the wrong version, the same list object is reused across every call that does not provide a records argument. The list accumulates values across calls, which is almost never what you want. The correct pattern is to default to None and create a new list inside the function. You will see this pattern constantly in Python codebases.

**`*args` — variable number of positional arguments**

```python
def log_messages(*messages):
    for message in messages:
        print(f"[LOG] {message}")

log_messages("Pipeline started")
log_messages("Reading file", "Transforming data", "Writing output")
```

`*args` collects all extra positional arguments into a tuple. The name `args` is a convention — the `*` is what matters. Inside the function, `messages` is a tuple you can iterate over. Useful when you do not know in advance how many arguments will be passed.

**`**kwargs` — variable number of keyword arguments**

```python
def create_connection(**config):
    for key, value in config.items():
        print(f"{key}: {value}")

create_connection(host="localhost", port=5432, database="learning", user="postgres")
```

`**kwargs` collects all extra keyword arguments into a dictionary. Inside the function, `config` is a regular dictionary. This pattern appears constantly in data engineering libraries — database connectors, cloud SDK clients, and pipeline frameworks all use `**kwargs` to accept flexible configuration.

You can combine all of these. The order must always be: positional, `*args`, keyword with defaults, `**kwargs`.

```python
def pipeline_run(name, *sources, output_format="parquet", **options):
    print(f"Running pipeline: {name}")
    print(f"Sources: {sources}")
    print(f"Output format: {output_format}")
    print(f"Options: {options}")

pipeline_run(
    "sales_pipeline",
    "orders.csv", "customers.csv",
    output_format="csv",
    overwrite=True,
    batch_size=1000
)
```

---

### Return values — what functions actually give back

A function can return any Python object — a number, a string, a list, a dictionary, another function, or nothing.

```python
def get_file_stats(filepath):
    # pretending to read a file
    row_count = 1000
    column_count = 12
    size_mb = 4.7
    return row_count, column_count, size_mb

rows, cols, size = get_file_stats("data.csv")
print(rows, cols, size)
```

Returning multiple values with commas actually returns a tuple. The unpacking on the left side (`rows, cols, size`) splits the tuple into individual variables. This is called tuple unpacking and you will see it constantly.

A function with no return statement returns `None` implicitly. This is Python's equivalent of `void` in Java — but unlike Java, you do not declare it. If your function produces a side effect (writes a file, prints output, sends a message) rather than computing a value, it is fine to return nothing. But in data engineering, functions that transform data should almost always return their result rather than modifying things in place — it makes them easier to test and chain together.

---

### Scope — where variables live

```python
rows_processed = 0  # module-level variable

def process_batch(batch_size):
    rows_processed = batch_size  # local variable -- different from the one above
    print(f"Processed {rows_processed} rows")

process_batch(500)
print(rows_processed)  # prints 0 -- the module-level variable is unchanged
```

Variables defined inside a function are local — they exist only while the function runs and disappear when it returns. They do not affect variables with the same name outside the function.

This is identical to Java's concept of local scope. The key difference is that Python does not require you to declare variables — you just assign them. The first assignment inside a function creates a local variable.

To modify a module-level variable from inside a function, you need the `global` keyword. In data engineering code, you almost never want to do this. Global mutable state makes pipelines unpredictable and untestable. If a function needs to communicate a result back to the caller, it should return it.

---

### Type hints — making Python code readable and safe

Python does not enforce types at runtime by default. You can pass an integer where a string is expected and Python will not complain until the code breaks. This is a significant problem in large data engineering codebases where a function is called from dozens of places.

Type hints solve this:

```python
def load_csv(filepath: str, delimiter: str = ",", skip_rows: int = 0) -> list[dict]:
    """
    Load a CSV file and return a list of row dictionaries.
    
    Args:
        filepath: Path to the CSV file
        delimiter: Column separator character
        skip_rows: Number of header rows to skip
    
    Returns:
        List of dictionaries where each dict represents one row
    """
    pass  # implementation comes later
```

The `: str` after each parameter name is the type hint. The `-> list[dict]` after the closing parenthesis is the return type hint. Python does not enforce these at runtime — a wrong type will not raise an error unless you run a type checker like mypy separately. But they serve three critical purposes.

First, they document what the function expects and returns, which is more reliable than a comment because the hints stay next to the code they describe.

Second, VS Code reads type hints and uses them for autocomplete and inline error detection. If you type `load_csv(filepath=123)` in VS Code, it underlines the integer with a warning.

Third, in team environments, a type checker like mypy is run as part of CI/CD to catch type mismatches before code reaches production. You will encounter this in the production engineering phase.

The docstring — the triple-quoted string immediately after the function definition — is the standard way to document what a function does, what its parameters mean, and what it returns. In data engineering codebases that get maintained by teams, every function has a docstring. Build this habit from the start.

---

### Functions as first-class objects — a concept Java makes difficult

In Python, functions are objects. You can assign them to variables, pass them as arguments to other functions, and return them from functions. This is called first-class functions and it unlocks patterns that are central to how data engineering code is written.

```python
def double(x: int) -> int:
    return x * 2

def square(x: int) -> int:
    return x * 2

def apply_transformation(values: list, transform_function):
    return [transform_function(v) for v in values]

numbers = [1, 2, 3, 4, 5]
print(apply_transformation(numbers, double))   # [2, 4, 6, 8, 10]
print(apply_transformation(numbers, square))   # [1, 4, 9, 16, 25]
```

`apply_transformation` takes a function as an argument and applies it to each element. You passed `double` and `square` as arguments — no parentheses, because parentheses would call the function immediately. Without parentheses you are passing the function object itself.

In data engineering, this pattern appears in pipeline frameworks constantly. You define a transformation function, then pass it to a pipeline runner that applies it to each batch of data. The pipeline runner does not know or care what transformation you pass — it just calls whatever function you gave it.

---

### Lambda functions — small anonymous functions

```python
numbers = [1, 2, 3, 4, 5]
doubled = list(map(lambda x: x * 2, numbers))
print(doubled)  # [2, 4, 6, 8, 10]
```

A lambda is a function without a name, defined inline with a single expression. `lambda x: x * 2` is equivalent to:

```python
def anonymous(x):
    return x * 2
```

Lambdas are limited — they can only contain a single expression, no statements, no multiple lines. They exist for short, throwaway transformations that are not worth naming.

In data engineering, you will see lambdas used with Pandas `apply`, Spark's `map` and `filter`, and sorting operations:

```python
records = [
    {"name": "Arjun", "gpa": 8.5},
    {"name": "Priya", "gpa": 9.1},
    {"name": "Rohan", "gpa": 7.8},
]

sorted_records = sorted(records, key=lambda r: r["gpa"], reverse=True)
```

`key=lambda r: r["gpa"]` tells the sort function how to extract the comparison value from each element. This is the first-class function pattern again — you are passing a function as an argument.

For anything longer than one line or used more than once, define a proper named function instead of a lambda. Lambdas are for brevity in simple cases, not for complex logic.

---

### Nested functions and closures — functions that remember their environment

```python
def make_multiplier(factor: int):
    def multiply(x: int) -> int:
        return x * factor
    return multiply

triple = make_multiplier(3)
print(triple(5))   # 15
print(triple(10))  # 30

double = make_multiplier(2)
print(double(5))   # 10
```

`make_multiplier` returns a function. The returned `multiply` function remembers the value of `factor` from when it was created — even after `make_multiplier` has finished executing. This memory of the enclosing scope is called a **closure**.

`triple` is now a function that multiplies by 3. `double` is a different function that multiplies by 2. Same inner function definition, different captured environment.

In data engineering, closures appear when you need to configure a transformation function at pipeline setup time and use it later during execution. You create a function that captures configuration values and returns a configured transformation:

```python
def make_cleaner(null_replacement: str, strip_whitespace: bool):
    def clean(value: str) -> str:
        if value is None:
            return null_replacement
        if strip_whitespace:
            value = value.strip()
        return value
    return clean

clean_strict = make_cleaner(null_replacement="UNKNOWN", strip_whitespace=True)
clean_lenient = make_cleaner(null_replacement="", strip_whitespace=False)
```

---

### Decorators — functions that wrap other functions

A decorator is a function that takes a function as input and returns a modified version of that function. The `@` syntax is syntactic sugar for applying a decorator.

```python
def log_execution(func):
    def wrapper(*args, **kwargs):
        print(f"Starting: {func.__name__}")
        result = func(*args, **kwargs)
        print(f"Finished: {func.__name__}")
        return result
    return wrapper

@log_execution
def extract_data(source: str) -> list:
    print(f"Extracting from {source}")
    return [1, 2, 3]

data = extract_data("database")
```

Output:

```
Starting: extract_data
Extracting from database
Finished: extract_data
```

`@log_execution` above the function definition is equivalent to writing `extract_data = log_execution(extract_data)` after the definition. The decorator replaces the original function with the wrapper version that adds logging before and after.

In data engineering, decorators are used for:

- Logging when pipeline steps start and finish
- Measuring how long each step takes
- Retrying a function automatically if it raises an exception (critical for network calls that may fail transiently)
- Caching expensive function results

You will encounter decorators heavily in Airflow — every task in an Airflow pipeline is defined using decorators in the modern TaskFlow API. Understanding what a decorator does underneath makes Airflow's syntax obvious rather than magical.

---

### Error handling inside functions — a data engineering necessity

In Java you were likely taught try-catch. Python uses try-except with the same concept.

```python
def read_file(filepath: str) -> str:
    try:
        with open(filepath, "r", encoding="utf-8") as f:
            return f.read()
    except FileNotFoundError:
        print(f"File not found: {filepath}")
        return None
    except PermissionError:
        print(f"No permission to read: {filepath}")
        return None
    except Exception as e:
        print(f"Unexpected error reading {filepath}: {e}")
        raise
```

Three important patterns here.

First, catch specific exceptions before general ones. `FileNotFoundError` is caught specifically because you know exactly how to handle it — return None and log a message. The bare `except Exception as e` at the bottom is a catch-all for anything unexpected — and critically, it re-raises with `raise` instead of silently swallowing the error. In data engineering, silently swallowing unexpected errors is catastrophic — your pipeline appears to succeed while producing wrong or missing data.

Second, the `with open(...)` syntax. This is a context manager — it guarantees the file is closed when the block exits, even if an exception occurs. We will cover context managers formally in Lesson 3, but start using `with` for file operations immediately.

Third, the function returns `None` on known failure cases. The caller must check for None before using the result. A better pattern in production code is to raise a specific exception rather than return None — it forces the caller to handle the failure explicitly rather than accidentally using a None value. We will build on this in the error handling lesson.

---

### Exercises — Lesson 1

These exercises connect directly to data engineering work. The answers follow.

1. Write a function `clean_string` that takes a string, strips leading and trailing whitespace, converts it to lowercase, and replaces any internal sequences of multiple spaces with a single space. It should have a type hint for the parameter and return value. Test it with `" Hello World "`.
    
2. Write a function `calculate_stats` that accepts any number of numeric values using `*args` and returns a dictionary containing the count, sum, minimum, maximum, and average. Handle the case where no values are passed.
    
3. Write a function `make_validator` that accepts a minimum value and maximum value and returns a new function. The returned function should accept a number and return True if it is within the range (inclusive) and False otherwise. Use this to create a `valid_gpa` validator and test it.
    
4. Write a decorator `timer` that measures and prints how long a function takes to execute in milliseconds. Apply it to a function that does a simple computation. Use Python's `time` module — specifically `time.time()` which returns the current time in seconds as a float.
    
5. Write a function `safe_divide` that divides two numbers. If the divisor is zero, it should not crash — instead it should raise a `ValueError` with a descriptive message. Write it with proper type hints and a docstring. Test both the normal case and the zero case.
    

---

### Answers — Lesson 1

**Exercise 1**

```python
import re

def clean_string(text: str) -> str:
    """
    Clean a string by stripping whitespace, lowercasing,
    and collapsing internal spaces.
    """
    text = text.strip()
    text = text.lower()
    text = re.sub(r' +', ' ', text)
    return text

print(clean_string("  Hello   World  "))  # "hello world"
```

**Exercise 2**

```python
def calculate_stats(*values: float) -> dict:
    """
    Compute basic statistics for any number of numeric values.
    Returns empty dict if no values provided.
    """
    if not values:
        return {}
    return {
        "count": len(values),
        "sum": sum(values),
        "min": min(values),
        "max": max(values),
        "average": sum(values) / len(values),
    }

print(calculate_stats(8.5, 9.1, 7.8, 6.5, 9.4))
print(calculate_stats())
```

**Exercise 3**

```python
def make_validator(min_value: float, max_value: float):
    """Return a function that checks if a value is within range."""
    def validate(value: float) -> bool:
        return min_value <= value <= max_value
    return validate

valid_gpa = make_validator(0.0, 10.0)
print(valid_gpa(8.5))   # True
print(valid_gpa(11.0))  # False
print(valid_gpa(-1.0))  # False
```

**Exercise 4**

```python
import time

def timer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        elapsed_ms = (end - start) * 1000
        print(f"{func.__name__} took {elapsed_ms:.2f}ms")
        return result
    return wrapper

@timer
def sum_large_range(n: int) -> int:
    return sum(range(n))

result = sum_large_range(10_000_000)
print(f"Result: {result}")
```

**Exercise 5**

```python
def safe_divide(numerator: float, denominator: float) -> float:
    """
    Divide numerator by denominator.

    Args:
        numerator: The number to be divided
        denominator: The number to divide by

    Returns:
        Result of the division

    Raises:
        ValueError: If denominator is zero
    """
    if denominator == 0:
        raise ValueError(
            f"Cannot divide {numerator} by zero"
        )
    return numerator / denominator

print(safe_divide(10, 2))    # 5.0

try:
    print(safe_divide(10, 0))
except ValueError as e:
    print(f"Caught error: {e}")
```

---

## Phase 2, Lesson 2 — Object Oriented Programming in Python

---

### Why OOP matters in data engineering

You already understand OOP from Java — classes, objects, inheritance, interfaces. The concepts are identical in Python. What changes is the syntax, some conventions, and a few Python-specific features that you will encounter constantly in data engineering libraries.

The reason OOP matters specifically in data engineering: the systems you build — pipeline runners, data connectors, transformation engines, schema validators — are complex enough that they need to be modeled as objects with state and behavior. A database connection is an object. A pipeline configuration is an object. A data schema is an object. Every major data engineering library you will use — SQLAlchemy, Pandas, Pydantic, Airflow — is built around classes.

---

### Defining a class — syntax and conventions

```python
class DataSource:
    """Represents a data source that can be read from."""

    def __init__(self, name: str, filepath: str, delimiter: str = ","):
        self.name = name
        self.filepath = filepath
        self.delimiter = delimiter
        self.row_count = 0
        self._is_loaded = False  # private by convention

    def load(self) -> list[dict]:
        """Load data from the source file."""
        print(f"Loading {self.name} from {self.filepath}")
        self._is_loaded = True
        return []

    def get_status(self) -> str:
        """Return the current load status."""
        if self._is_loaded:
            return f"{self.name}: loaded ({self.row_count} rows)"
        return f"{self.name}: not loaded"


source = DataSource("sales", "sales.csv")
print(source.get_status())
source.load()
print(source.get_status())
```

`__init__` is the constructor — equivalent to Java's constructor method. It runs when you create an object. `self` is the reference to the current object — equivalent to Java's `this`. Unlike Java, you must explicitly include `self` as the first parameter of every instance method. Python passes it automatically when you call the method, but you must declare it.

The single underscore prefix on `_is_loaded` is a Python convention meaning "this is intended to be private — do not access it from outside the class." Python does not enforce this — you technically can access `source._is_loaded` from outside. The underscore is a signal to other developers. This is different from Java's `private` keyword which is enforced by the compiler.

---

### Class variables vs instance variables

```python
class Pipeline:
    
    pipeline_count = 0  # class variable -- shared across all instances

    def __init__(self, name: str):
        Pipeline.pipeline_count += 1
        self.name = name           # instance variable -- unique per object
        self.run_count = 0         # instance variable -- unique per object

p1 = Pipeline("sales_pipeline")
p2 = Pipeline("inventory_pipeline")

print(Pipeline.pipeline_count)  # 2
print(p1.name)                  # sales_pipeline
print(p2.name)                  # inventory_pipeline
```

Class variables are defined directly in the class body, outside any method. They are shared across all instances. Instance variables are defined inside `__init__` with `self.` and are unique to each object.

In data engineering, class variables are useful for tracking things at the class level — how many connections have been opened, what the default configuration is, a registry of all registered pipeline types.

---

### Inheritance — reusing and extending behavior

```python
class DataSource:
    def __init__(self, name: str):
        self.name = name

    def connect(self) -> bool:
        raise NotImplementedError("Subclasses must implement connect()")

    def read(self) -> list[dict]:
        raise NotImplementedError("Subclasses must implement read()")

    def describe(self) -> str:
        return f"DataSource: {self.name}"


class CSVSource(DataSource):
    def __init__(self, name: str, filepath: str, delimiter: str = ","):
        super().__init__(name)          # call parent constructor
        self.filepath = filepath
        self.delimiter = delimiter

    def connect(self) -> bool:
        print(f"Checking if {self.filepath} exists")
        return True

    def read(self) -> list[dict]:
        print(f"Reading CSV from {self.filepath}")
        return []

    def describe(self) -> str:
        base = super().describe()       # call parent method
        return f"{base} (CSV at {self.filepath})"


class DatabaseSource(DataSource):
    def __init__(self, name: str, host: str, database: str):
        super().__init__(name)
        self.host = host
        self.database = database

    def connect(self) -> bool:
        print(f"Connecting to {self.database} at {self.host}")
        return True

    def read(self) -> list[dict]:
        print(f"Querying {self.database}")
        return []


csv_source = CSVSource("sales", "sales.csv")
db_source = DatabaseSource("customers", "localhost", "production")

sources = [csv_source, db_source]
for source in sources:
    source.connect()    # calls the correct version for each type
    print(source.describe())
```

`super().__init__(name)` calls the parent class constructor. In Java you write `super(name)`. In Python it is `super().__init__(name)`. The pattern is the same.

The `for source in sources` loop demonstrates **polymorphism** — the same `connect()` call works on both types because they both inherit from DataSource and both implement connect(). The correct version is called based on the actual type of the object at runtime. Identical to Java's polymorphism.

`raise NotImplementedError` in the base class is Python's equivalent of an abstract method in Java. Python has a formal way to do this using the `abc` module, which we cover next.

---

### Abstract base classes — enforcing the interface contract

```python
from abc import ABC, abstractmethod

class DataSource(ABC):
    def __init__(self, name: str):
        self.name = name

    @abstractmethod
    def connect(self) -> bool:
        """Establish connection to the data source."""
        pass

    @abstractmethod
    def read(self) -> list[dict]:
        """Read and return data as a list of dictionaries."""
        pass

    def describe(self) -> str:
        return f"DataSource: {self.name}"


class CSVSource(DataSource):
    def __init__(self, name: str, filepath: str):
        super().__init__(name)
        self.filepath = filepath

    def connect(self) -> bool:
        return True

    def read(self) -> list[dict]:
        return []
```

`ABC` (Abstract Base Class) and `@abstractmethod` are Python's equivalents of Java's `abstract class` and `abstract method`. If you try to instantiate `DataSource` directly, Python raises a `TypeError`. If a subclass does not implement all abstract methods, Python raises a `TypeError` when you try to instantiate the subclass. This is the same enforcement Java gives you with abstract methods, just at instantiation time rather than compile time.

This pattern is fundamental to how data engineering frameworks work. Airflow's operators inherit from a base operator class and implement specific abstract methods. SQLAlchemy's database dialects inherit from a base dialect and implement connection methods. Every connector, operator, and plugin system in data engineering uses this pattern. Understanding it here means every framework you encounter later will feel familiar.

---

### Special methods — making classes behave like Python built-ins

Python classes can implement special methods (called dunder methods — double underscore on both sides) that let your objects work with Python's built-in operations.

```python
class DataSchema:
    def __init__(self, name: str, columns: dict[str, str]):
        self.name = name
        self.columns = columns  # {"column_name": "data_type"}

    def __repr__(self) -> str:
        return f"DataSchema(name={self.name!r}, columns={len(self.columns)})"

    def __str__(self) -> str:
        cols = "\n".join(f"  {col}: {dtype}" for col, dtype in self.columns.items())
        return f"Schema '{self.name}':\n{cols}"

    def __len__(self) -> int:
        return len(self.columns)

    def __contains__(self, column_name: str) -> bool:
        return column_name in self.columns

    def __eq__(self, other) -> bool:
        if not isinstance(other, DataSchema):
            return False
        return self.name == other.name and self.columns == other.columns


schema = DataSchema("students", {
    "student_id": "INTEGER",
    "name": "VARCHAR",
    "gpa": "DECIMAL"
})

print(schema)            # calls __str__
print(repr(schema))      # calls __repr__
print(len(schema))       # calls __len__ -- prints 3
print("gpa" in schema)   # calls __contains__ -- prints True
print("age" in schema)   # calls __contains__ -- prints False
```

`__repr__` should return an unambiguous string representation — useful for debugging. Convention is that `repr(obj)` should ideally return a string that could be used to recreate the object.

`__str__` should return a human-readable string — used by `print()`.

`__len__` makes `len(schema)` work.

`__contains__` makes the `in` operator work.

`__eq__` defines what equality means for your class. Without this, two different objects with identical data are not equal — Python compares object identity (memory address) by default, exactly like Java's `==` on objects.

In data engineering, you will implement these methods in custom classes that represent schemas, pipeline configurations, and data records. They make your code more Pythonic and integrate naturally with Python's built-in operations.

---

### Dataclasses — the modern way to define data-holding classes

Python 3.7 introduced dataclasses — a way to define classes that primarily hold data without writing boilerplate `__init__`, `__repr__`, and `__eq__` methods.

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class PipelineConfig:
    name: str
    source_path: str
    output_path: str
    delimiter: str = ","
    encoding: str = "utf-8"
    batch_size: int = 1000
    tags: list[str] = field(default_factory=list)
    max_retries: Optional[int] = None

    def is_valid(self) -> bool:
        return bool(self.name and self.source_path and self.output_path)


config = PipelineConfig(
    name="sales_pipeline",
    source_path="/data/sales.csv",
    output_path="/output/sales.parquet",
    batch_size=5000,
    tags=["sales", "daily"]
)

print(config)
print(config.is_valid())

config2 = PipelineConfig(
    name="sales_pipeline",
    source_path="/data/sales.csv",
    output_path="/output/sales.parquet"
)
print(config == config2)  # False -- different batch_size and tags
```

The `@dataclass` decorator automatically generates `__init__`, `__repr__`, and `__eq__` based on the class attributes and their type hints. `field(default_factory=list)` is the correct way to set a mutable default — it creates a new list for each instance, avoiding the mutable default argument trap from Lesson 1.

In data engineering, dataclasses are used heavily for configuration objects, schema definitions, pipeline metadata, and any class whose primary purpose is holding structured data. They are cleaner and less error-prone than writing the boilerplate manually.

---

### Exercises — Lesson 2

1. Define an abstract base class `DataWriter` with abstract methods `connect() -> bool` and `write(data: list[dict]) -> int` (returns the number of rows written). Implement two concrete subclasses — `CSVWriter` and `JSONWriter`. Each should print what it is doing when connect and write are called. CSVWriter should accept a filepath and delimiter. JSONWriter should accept a filepath and an indent level for pretty printing.
    
2. Create a dataclass `ColumnSchema` with fields: name (str), data_type (str), nullable (bool, default True), primary_key (bool, default False). Then create a dataclass `TableSchema` that holds a table name and a list of ColumnSchema objects. Add a method to TableSchema called `get_primary_keys()` that returns a list of column names that are primary keys. Add a `__len__` method that returns the number of columns.
    
3. Write a decorator called `retry` that accepts a parameter `max_attempts` and retries a function up to that many times if it raises an exception. It should print which attempt it is on. Apply it to a function that randomly fails — use Python's `random` module, `random.random() < 0.7` returns True 70% of the time, use this to simulate failure.
    

---

### Answers — Lesson 2

**Exercise 1**

```python
from abc import ABC, abstractmethod

class DataWriter(ABC):
    def __init__(self, filepath: str):
        self.filepath = filepath

    @abstractmethod
    def connect(self) -> bool:
        pass

    @abstractmethod
    def write(self, data: list[dict]) -> int:
        pass


class CSVWriter(DataWriter):
    def __init__(self, filepath: str, delimiter: str = ","):
        super().__init__(filepath)
        self.delimiter = delimiter

    def connect(self) -> bool:
        print(f"Opening CSV file for writing: {self.filepath}")
        return True

    def write(self, data: list[dict]) -> int:
        print(f"Writing {len(data)} rows to CSV with delimiter '{self.delimiter}'")
        return len(data)


class JSONWriter(DataWriter):
    def __init__(self, filepath: str, indent: int = 2):
        super().__init__(filepath)
        self.indent = indent

    def connect(self) -> bool:
        print(f"Opening JSON file for writing: {self.filepath}")
        return True

    def write(self, data: list[dict]) -> int:
        print(f"Writing {len(data)} records to JSON with indent={self.indent}")
        return len(data)


writers = [
    CSVWriter("/output/data.csv", delimiter=";"),
    JSONWriter("/output/data.json", indent=4),
]

sample_data = [{"id": 1, "name": "Arjun"}, {"id": 2, "name": "Priya"}]

for writer in writers:
    writer.connect()
    rows_written = writer.write(sample_data)
    print(f"Wrote {rows_written} rows\n")
```

**Exercise 2**

```python
from dataclasses import dataclass, field

@dataclass
class ColumnSchema:
    name: str
    data_type: str
    nullable: bool = True
    primary_key: bool = False


@dataclass
class TableSchema:
    table_name: str
    columns: list[ColumnSchema] = field(default_factory=list)

    def get_primary_keys(self) -> list[str]:
        return [col.name for col in self.columns if col.primary_key]

    def __len__(self) -> int:
        return len(self.columns)


students_schema = TableSchema(
    table_name="students",
    columns=[
        ColumnSchema("student_id", "INTEGER", nullable=False, primary_key=True),
        ColumnSchema("name", "VARCHAR", nullable=False),
        ColumnSchema("city", "VARCHAR"),
        ColumnSchema("gpa", "DECIMAL"),
    ]
)

print(students_schema)
print(f"Number of columns: {len(students_schema)}")
print(f"Primary keys: {students_schema.get_primary_keys()}")
```

**Exercise 3**

```python
import random
import time

def retry(max_attempts: int):
    def decorator(func):
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    print(f"Attempt {attempt} of {max_attempts}")
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == max_attempts:
                        print(f"All {max_attempts} attempts failed")
                        raise
                    print(f"Attempt {attempt} failed: {e}. Retrying...")
        return wrapper
    return decorator


@retry(max_attempts=5)
def unstable_api_call() -> str:
    if random.random() < 0.7:
        raise ConnectionError("API temporarily unavailable")
    return "Success"


result = unstable_api_call()
print(result)
```

---

## Phase 2, Lesson 3 — File I/O and Data Formats

---

### Why this lesson exists in a data engineering curriculum

A data engineer spends an enormous amount of time reading and writing files. Not because files are the ideal storage format for large-scale data — they are not — but because files are the universal interchange format between systems. Data arrives as CSV exports from databases, JSON responses from APIs, Parquet files from data lakes, Avro files from Kafka streams. You need to handle all of them fluently.

More importantly, the choices you make about file formats have direct consequences on pipeline performance and storage costs. A CSV file of 1GB uncompressed might be 80MB as Parquet with better query performance. Understanding why leads to better architectural decisions.

---

### Context managers — the right way to handle files

Before we open any file, you need to understand context managers properly because you will use them for every file operation, every database connection, and every network call in your career.

```python
# Wrong -- file may not be closed if an exception occurs
f = open("data.txt", "r")
content = f.read()
f.close()

# Correct -- file is guaranteed to close even if an exception occurs
with open("data.txt", "r", encoding="utf-8") as f:
    content = f.read()
```

The `with` statement calls `__enter__` on the context manager when entering the block and `__exit__` when leaving — whether the block exits normally or via an exception. For files, `__exit__` closes the file. You never need to call `f.close()` manually when using `with`.

This matters in data engineering because pipelines run unattended. If a file is not closed after an exception, you get resource leaks — too many open file handles — which can crash a long-running pipeline hours into execution. Always use `with` for files, database connections, and any other resource that needs explicit cleanup.

You can implement context managers in your own classes using `__enter__` and `__exit__`:

```python
class DatabaseConnection:
    def __init__(self, host: str, database: str):
        self.host = host
        self.database = database
        self.connection = None

    def __enter__(self):
        print(f"Connecting to {self.database}")
        self.connection = "fake_connection_object"
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print(f"Closing connection to {self.database}")
        self.connection = None
        return False  # False means do not suppress exceptions

    def query(self, sql: str) -> list:
        print(f"Executing: {sql}")
        return []


with DatabaseConnection("localhost", "production") as db:
    results = db.query("SELECT * FROM students")
# Connection is closed here automatically
```

`__exit__` receives information about any exception that occurred. Returning False means the exception propagates normally. Returning True would suppress the exception — almost never what you want.

---

### Reading and writing text files

```python
# Writing a text file
with open("output.txt", "w", encoding="utf-8") as f:
    f.write("First line\n")
    f.write("Second line\n")

# Appending to a file
with open("output.txt", "a", encoding="utf-8") as f:
    f.write("Third line\n")

# Reading entire file at once
with open("output.txt", "r", encoding="utf-8") as f:
    content = f.read()
    print(content)

# Reading line by line -- memory efficient for large files
with open("output.txt", "r", encoding="utf-8") as f:
    for line in f:
        print(line.strip())  # strip() removes the trailing newline
```

The file mode matters:

- `"r"` — read. Raises FileNotFoundError if file does not exist.
- `"w"` — write. Creates the file if it does not exist. **Overwrites** existing content.
- `"a"` — append. Creates the file if it does not exist. Adds to end of existing content.
- `"rb"`, `"wb"` — binary read/write. Used for non-text files.

Always specify `encoding="utf-8"` explicitly. Without it, Python uses the system default encoding which varies between Windows, Mac, and Linux. A pipeline that works on your Windows machine may fail on a Linux server in the cloud due to encoding differences. Be explicit.

For large files, never read the entire file into memory with `f.read()`. A 10GB file will consume 10GB of RAM and likely crash your process. Always iterate line by line with `for line in f` — Python reads one line at a time, keeping memory usage constant regardless of file size.

---

### CSV — the most common data format you will encounter

CSV (Comma Separated Values) is the universal interchange format. Every database, spreadsheet, and analytics tool can export and import CSV. It is also the worst format for large-scale data engineering — no schema, no types, slow to parse, verbose. But you will deal with it constantly because it is everywhere.

```python
import csv

# Writing CSV
rows = [
    {"student_id": 1, "name": "Arjun Mehta", "gpa": 8.5},
    {"student_id": 2, "name": "Priya Sharma", "gpa": 9.1},
    {"student_id": 3, "name": "Rohan Verma", "gpa": 7.8},
]

with open("students.csv", "w", newline="", encoding="utf-8") as f:
    writer = csv.DictWriter(f, fieldnames=["student_id", "name", "gpa"])
    writer.writeheader()
    writer.writerows(rows)

# Reading CSV
with open("students.csv", "r", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(dict(row))
```

`newline=""` in the write mode is required on Windows. Without it, Python adds an extra carriage return on each line, producing files with blank lines between rows. This is a Windows-specific quirk.

`DictWriter` and `DictReader` are almost always preferable to the basic `writer` and `reader` — they work with dictionaries keyed by column name rather than positional lists, which makes code more readable and resilient to column reordering.

Notice that everything read from a CSV is a string. `row["student_id"]` is the string `"1"`, not the integer `1`. `row["gpa"]` is the string `"8.5"`, not the float `8.5`. You must convert types explicitly. This is one of CSV's fundamental limitations — it has no type information.

```python
def parse_student_row(row: dict) -> dict:
    return {
        "student_id": int(row["student_id"]),
        "name": row["name"].strip(),
        "gpa": float(row["gpa"]) if row["gpa"] else None,
    }

with open("students.csv", "r", encoding="utf-8") as f:
    reader = csv.DictReader(f)
    students = [parse_student_row(row) for row in reader]
```

The type conversion step — parsing raw strings from CSV into proper Python types — is something every data pipeline that reads CSV must do. Forgetting it leads to silent bugs where numeric operations are performed on strings. In Python, `"8.5" + "9.1"` is `"8.59.1"`, not `17.6`.

---

### JSON — for APIs and semi-structured data

JSON (JavaScript Object Notation) is the standard format for API responses and configuration files. Unlike CSV, JSON preserves basic type information — numbers are numbers, strings are strings, booleans are booleans, null is null.

```python
import json

# Python dict/list to JSON string
data = {
    "pipeline": "sales_etl",
    "run_date": "2024-01-15",
    "stats": {
        "rows_processed": 15420,
        "errors": 0,
        "duration_seconds": 47.3
    },
    "tables": ["orders", "customers", "products"]
}

# Write JSON to file
with open("run_metadata.json", "w", encoding="utf-8") as f:
    json.dump(data, f, indent=2)

# Read JSON from file
with open("run_metadata.json", "r", encoding="utf-8") as f:
    loaded = json.load(f)
    print(loaded["stats"]["rows_processed"])

# JSON string to Python object (for API responses)
json_string = '{"status": "success", "rows": 100}'
parsed = json.loads(json_string)
print(parsed["status"])

# Python object to JSON string
back_to_string = json.dumps(data, indent=2)
```

`json.dump` — write to file. `json.dumps` — convert to string (the s stands for string).  
`json.load` — read from file. `json.loads` — parse from string.

JSON handles nested structures natively — dictionaries, lists, nested combinations. This makes it suitable for semi-structured data where each record might have different fields. But it is verbose (field names repeated on every record) and slow to parse at scale compared to binary formats.

One common trap: JSON does not support Python's `datetime` objects, `Decimal` types, or custom objects by default. Trying to serialize these raises a `TypeError`. You need a custom encoder:

```python
import json
from datetime import datetime
from decimal import Decimal

class DataEngineeringEncoder(json.JSONEncoder):
    def default(self, obj):
        if isinstance(obj, datetime):
            return obj.isoformat()
        if isinstance(obj, Decimal):
            return float(obj)
        return super().default(obj)

data = {
    "timestamp": datetime.now(),
    "amount": Decimal("1234.56")
}

print(json.dumps(data, cls=DataEngineeringEncoder, indent=2))
```

You will encounter this when serializing pipeline metadata, database query results, or any data that comes from PostgreSQL — which returns Decimal types for numeric columns.

---

### Parquet — the format you will actually use in production

CSV and JSON are human-readable but inefficient for large-scale data. Parquet is a binary columnar format designed specifically for analytical workloads. It is the standard file format in modern data engineering.

**Why columnar storage matters.** In a row-based format like CSV, a file with 10 columns stores all 10 column values for row 1, then all 10 for row 2, and so on. If a query only needs 2 columns out of 10, you still read all 10 columns worth of data for every row. In a columnar format like Parquet, all values for column 1 are stored together, all values for column 2 together, and so on. A 2-column query reads only those 2 columns' data — potentially 80% less I/O.

**Why compression works better on columnar data.** Column values tend to be of the same type and similar magnitude — all ages between 18 and 25, all city names from a small set. This repetition compresses far more efficiently than interleaved row data. A 1GB CSV file commonly becomes 50-100MB as Parquet with default compression.

```python
import pyarrow as pa
import pyarrow.parquet as pq

# Install first: pip install pyarrow

# Create data as a PyArrow table
data = {
    "student_id": [1, 2, 3, 4, 5],
    "name": ["Arjun", "Priya", "Rohan", "Sneha", "Karan"],
    "gpa": [8.5, 9.1, 7.8, 8.9, 6.5],
    "city": ["Mumbai", "Delhi", "Mumbai", "Chennai", "Ahmedabad"]
}

schema = pa.schema([
    pa.field("student_id", pa.int32()),
    pa.field("name", pa.string()),
    pa.field("gpa", pa.float32()),
    pa.field("city", pa.string()),
])

table = pa.table(data, schema=schema)

# Write Parquet file
pq.write_table(table, "students.parquet", compression="snappy")

# Read Parquet file
loaded_table = pq.read_table("students.parquet")
print(loaded_table.to_pydict())

# Read only specific columns -- this is where Parquet shines
names_only = pq.read_table("students.parquet", columns=["name", "gpa"])
print(names_only.to_pydict())
```

The explicit schema definition is important. Unlike CSV, Parquet embeds the schema in the file. When the file is read later — even by a different system, a different programming language, or a different team — the types are preserved exactly. `student_id` will always be an int32, never a string.

`compression="snappy"` applies Snappy compression — a fast compression algorithm that prioritizes decompression speed over compression ratio. Snappy is the most common choice for data engineering workloads where read performance matters more than maximum compression. `gzip` gives better compression at the cost of slower reads. `zstd` is a modern alternative that balances both well.

Install pyarrow before running this:

```
pip install pyarrow
```

---

### Parquet partitioning — how large datasets are organized on disk

When you have millions of rows, a single Parquet file becomes unwieldy. Partitioning splits the data into multiple files organized by one or more column values.

```python
import pyarrow as pa
import pyarrow.parquet as pq

data = {
    "student_id": [1, 2, 3, 4, 5, 6, 7, 8],
    "name": ["Arjun", "Priya", "Rohan", "Sneha", "Karan", "Divya", "Amit", "Meera"],
    "city": ["Mumbai", "Delhi", "Mumbai", "Chennai", "Ahmedabad", "Delhi", "Pune", "Hyderabad"],
    "gpa": [8.5, 9.1, 7.8, 8.9, 6.5, 9.4, 7.2, 8.1]
}

table = pa.table(data)

# Write partitioned by city
pq.write_to_dataset(
    table,
    root_path="students_partitioned",
    partition_cols=["city"]
)

# Read only Mumbai partition -- reads only the Mumbai file
mumbai = pq.read_table(
    "students_partitioned",
    filters=[("city", "=", "Mumbai")]
)
print(mumbai.to_pydict())
```

This creates a directory structure:

```
students_partitioned/
  city=Mumbai/
    part-0.
```