
## Phase 2, Lesson 2 — Object Oriented Programming in Python

---

### Starting from the right place — what OOP actually solves

You know OOP from Java. But Java teaches OOP in a specific way — everything must be in a class, the compiler enforces structure, interfaces are explicit contracts. This can give the impression that OOP is about syntax and rules.

It is not. OOP is about one thing: **managing the relationship between data and the behavior that operates on that data.**

Before OOP, programs had data in one place and functions in another. The functions that were supposed to work on specific data had no formal connection to that data. As programs grew, this became chaotic — you had to know, by convention and memory, which functions worked on which data. There was no way for the language to help you.

OOP says: bundle the data and the behavior that belongs to it together into one unit. Call that unit an object. The object carries its own data and knows how to operate on it. The language can now enforce relationships and catch mistakes.

In Java this is mandatory — you cannot write code outside a class. In Python it is optional — you can write perfectly good Python code using only functions. But as your data engineering systems grow beyond a few hundred lines, the organizational benefits of OOP become essential. A database connection is not just a config dictionary and five functions that happen to use it — it is an object that owns its connection state and knows how to open, query, and close itself.

That is the why. Now let us understand how Python implements it, starting from what Python actually does in memory.

---

### What happens when Python executes a class definition

Before we write any useful class, you need to understand what Python does when it sees `class`. This is where Java and Python diverge significantly, and understanding the difference prevents enormous confusion.

In Java, a class is a compile-time construct. The compiler reads your class definition and produces bytecode. Classes exist before any program runs.

In Python, a class definition is executable code that runs at import time. When Python encounters:

```python
class Student:
    school = "IIT Delhi"
    
    def __init__(self, name):
        self.name = name
```

Python executes the class body as a block of code. It creates a new namespace, runs the body inside it, and then packages that namespace into a class object. `school` becomes an attribute of the class object. `__init__` becomes a function stored as an attribute of the class object.

After this executes, `Student` is a variable in the current scope that holds a class object. A class object is just another Python object — it has a type (`type`), it has attributes, it can be passed around as a value.

```python
print(type(Student))         # <class 'type'>
print(Student.school)        # IIT Delhi
print(Student.__init__)      # <function Student.__init__ at 0x...>
print(type(Student.__init__)) # <class 'function'>
```

The class object is not a blueprint that gets compiled. It is a live Python object that exists in memory and can be inspected, modified, and even created dynamically at runtime. This has deep consequences for how Python's object system works.

---

### Creating an instance — what actually happens

When you write `arjun = Student("Arjun Mehta")`, Python does this in sequence:

First, it calls `Student.__new__(Student)` — a method inherited from Python's base `object` class. This allocates a new, empty object in memory. The object knows it is an instance of `Student`, but has no attributes yet.

Second, it calls `Student.__init__(new_object, "Arjun Mehta")` — your constructor. The new object is passed as `self`. Your constructor code runs and adds attributes to it.

Third, the result is bound to `arjun`.

The object that `arjun` refers to is a container of attributes. When you assign `self.name = name` inside `__init__`, Python adds `"name"` to that container with value `"Arjun Mehta"`. The object's attribute namespace is a dictionary — you can actually see it:

```python
arjun = Student("Arjun Mehta")
print(arjun.__dict__)   # {"name": "Arjun Mehta"}
```

Every instance attribute you set in `__init__` ends up as an entry in `__dict__`. This is Python's actual storage mechanism for instance attributes — a plain dictionary.

When you access `arjun.name`, Python looks in `arjun.__dict__` first. If it finds `"name"` there, it returns that value. If it does not find it, it looks in the class object (`Student.__dict__`). If it finds it there, it returns that value. If not, it goes to the parent class, and so on up the inheritance chain.

This lookup order — instance dict, then class dict, then parent class dicts — is the mechanism behind both attribute access and method calls. Understanding it explains behaviors that otherwise seem magical.

---

### `self` — examined precisely

`self` is the most visually prominent thing in Python classes and the most commonly misunderstood.

In Java, `this` is an implicit keyword. The language automatically makes it available inside any instance method. You can reference it but you never declare it.

In Python, `self` must be explicitly declared as the first parameter of every instance method. Python does not make it implicitly available — it passes it as an explicit argument.

When you write:

```python
arjun.describe()
```

Python translates this to:

```python
Student.describe(arjun)
```

The object `arjun` is passed as the first argument to `describe`. Inside `describe`, that argument is named `self` by convention. So `self` is literally `arjun` — the same object, the same reference.

This means `self.name` is exactly the same as `arjun.name` when called as `arjun.describe()`. `self` is not a special reference to "the current object" in some abstract sense — it is the specific object the method was called on, passed as an ordinary argument.

The name `self` is a convention, not a keyword. You could name it `this` like Java, or `me`, or anything. But the entire Python community uses `self`, and deviating from it confuses every other Python developer who reads your code.

Why does Python make this explicit instead of implicit? It makes the mechanism transparent. There is no hidden state. When you see a method that starts with `self`, you immediately know it operates on the instance. When you see `self.something`, you know it is accessing the instance's attribute or method. Nothing is hidden.

---

### Instance attributes vs class attributes — the distinction examined deeply

This distinction looks simple on the surface but has subtle consequences in data engineering code.

```python
class Pipeline:
    default_batch_size = 1000      # class attribute
    pipeline_registry = {}         # class attribute -- DANGER

    def __init__(self, name: str, batch_size: int = None):
        self.name = name           # instance attribute
        self.batch_size = batch_size or Pipeline.default_batch_size
        self._run_count = 0        # instance attribute
```

**Class attributes** are defined in the class body, outside any method. There is one copy, stored in the class object's `__dict__`. Every instance shares this same copy. When you access `pipeline.default_batch_size`, Python does not find it in the instance's `__dict__`, so it looks up to the class's `__dict__` and finds it there.

**Instance attributes** are defined inside methods (typically `__init__`) by assigning to `self.something`. Each instance has its own copy in its own `__dict__`. `arjun.name` and `priya.name` are completely independent.

The deep consequence: **class attributes with mutable values are shared between all instances.**

```python
class Pipeline:
    pipeline_registry = {}   # one dictionary, shared by all instances

p1 = Pipeline("sales")
p2 = Pipeline("inventory")

Pipeline.pipeline_registry["sales"] = p1
print(p2.pipeline_registry)   # {"sales": p1} -- p2 sees p1's addition
```

This is sometimes intentional — a registry shared across all instances is a legitimate use of class attributes. But if you intended each pipeline to have its own registry, you made a serious mistake that will cause hard-to-debug bugs where one instance's state contaminates another's.

The correct way to give each instance its own mutable attribute:

```python
class Pipeline:
    def __init__(self, name: str):
        self.name = name
        self.pipeline_registry = {}   # new dict for each instance
```

Now each `Pipeline` instance has its own `pipeline_registry` in its own `__dict__`. Modifying one does not affect another.

The shadowing trap: if you assign to an instance attribute with the same name as a class attribute, Python creates a new instance attribute that shadows the class attribute for that instance only.

```python
class Pipeline:
    default_batch_size = 1000

p1 = Pipeline("sales")
p2 = Pipeline("inventory")

p1.default_batch_size = 500    # creates instance attribute on p1 only
print(p1.default_batch_size)   # 500 -- reads instance attribute
print(p2.default_batch_size)   # 1000 -- reads class attribute
print(Pipeline.default_batch_size)  # 1000 -- class is unchanged
```

`p1` now has `default_batch_size` in its own `__dict__`, shadowing the class attribute. `p2` has no `default_batch_size` in its `__dict__` so it still reads the class attribute. The class attribute is unchanged.

---

### Methods — all three kinds, examined from first principles

**Instance methods** — they operate on a specific object. The object is passed as `self`.

```python
class DataPipeline:
    def __init__(self, name: str, source: str):
        self.name = name
        self.source = source
        self._records = []
        self._run_count = 0

    def extract(self) -> list:
        """Pull records from the source."""
        print(f"Extracting from {self.source}")
        self._records = [{"id": 1}, {"id": 2}, {"id": 3}]
        return self._records

    def get_summary(self) -> str:
        return (
            f"Pipeline '{self.name}': "
            f"{len(self._records)} records, "
            f"{self._run_count} runs completed"
        )
```

Instance methods can read and modify the specific instance's attributes through `self`. They are the most common kind.

**Class methods** — they receive the class itself, not an instance. Decorated with `@classmethod`. The first parameter is conventionally named `cls`.

The primary use of class methods is **alternative constructors** — different ways to create an instance from different input formats.

```python
class DataPipeline:
    def __init__(self, name: str, source: str, batch_size: int = 1000):
        self.name = name
        self.source = source
        self.batch_size = batch_size
        self._records = []

    @classmethod
    def from_config_dict(cls, config: dict) -> "DataPipeline":
        """Create a pipeline from a configuration dictionary."""
        return cls(
            name=config["pipeline_name"],
            source=config["source_path"],
            batch_size=config.get("batch_size", 1000),
        )

    @classmethod
    def from_config_file(cls, filepath: str) -> "DataPipeline":
        """Create a pipeline from a JSON configuration file."""
        import json
        with open(filepath, "r", encoding="utf-8") as f:
            config = json.load(f)
        return cls.from_config_dict(config)
```

Why `cls` instead of `Pipeline` directly? Because subclasses inherit class methods. If `AdvancedPipeline` inherits from `DataPipeline` and calls `AdvancedPipeline.from_config_dict(config)`, `cls` is `AdvancedPipeline`, not `DataPipeline`. The factory creates an instance of the correct subclass. If you hardcoded `DataPipeline(...)` instead of `cls(...)`, calling the class method on a subclass would always create a `DataPipeline` instance, not the subclass. This is a subtle but important correctness issue.

The `-> "DataPipeline"` return type annotation is in quotes because at the point where the method is defined, the class `DataPipeline` is not yet fully defined. A string annotation is a forward reference — Python resolves it later. In Python 3.11+ you can use `from __future__ import annotations` at the top of the file to make all annotations strings automatically.

**Static methods** — they receive neither `self` nor `cls`. Decorated with `@staticmethod`. They are regular functions that live in the class namespace for organizational reasons.

```python
class DataPipeline:
    @staticmethod
    def validate_source_path(path: str) -> bool:
        """Check if a source path string is non-empty and has a file extension."""
        return bool(path) and "." in path.split("/")[-1]

    @staticmethod
    def sanitize_name(name: str) -> str:
        """Normalize a pipeline name to lowercase with underscores."""
        return name.strip().lower().replace(" ", "_").replace("-", "_")
```

Static methods cannot access class state or instance state. They are grouped in the class because they are logically related to the class, not because they need access to it.

The honest question to ask before writing a static method: does this function truly belong in this class, or is it a general utility function that happens to be related? If it could be used by completely unrelated parts of your codebase, it probably belongs as a standalone function in a utilities module, not as a static method. Overusing static methods is a sign of forcing OOP onto code that would be cleaner as plain functions.

---

### The `__init__` method in depth — what you are really doing

`__init__` is not a constructor in the Java sense. In Java, the constructor creates the object. In Python, `__new__` creates the object — `__init__` only initializes it. The object already exists when `__init__` runs. `self` inside `__init__` is the already-created but empty object.

This distinction matters in advanced scenarios (custom `__new__` implementations, metaclasses) but the practical implication for everyday code is this: `__init__` should do one thing — initialize the object's attributes to their starting state. It should not perform complex computations, make network connections, read files, or do anything that can fail in ways that leave a half-initialized object.

```python
# Bad -- __init__ does too much, can leave half-initialized objects
class DatabaseSource:
    def __init__(self, host: str, database: str):
        self.host = host
        self.database = database
        self.connection = self._connect()   # can fail
        self.schema = self._load_schema()   # can fail if connect failed

# Good -- __init__ only initializes, explicit method for setup
class DatabaseSource:
    def __init__(self, host: str, database: str):
        self.host = host
        self.database = database
        self.connection = None              # not connected yet
        self.schema = None                  # not loaded yet

    def connect(self) -> bool:
        """Explicitly establish the connection."""
        self.connection = f"connection_to_{self.database}"
        return True

    def load_schema(self) -> dict:
        """Explicitly load the schema after connecting."""
        if self.connection is None:
            raise RuntimeError("Must connect before loading schema")
        self.schema = {"student_id": "INTEGER", "name": "VARCHAR"}
        return self.schema
```

The second design gives the caller control over when expensive operations happen. It also makes partial initialization explicit — you know the object is connected only after calling `connect()`. And it makes each step testable independently.

---

### Inheritance — the mechanism examined carefully

Inheritance lets a child class reuse and extend the behavior of a parent class. The concept is identical to Java. The mechanism has important Python-specific details.

```python
class DataSource:
    """Base class for all data sources."""

    def __init__(self, name: str):
        self.name = name
        self._connected = False
        self._records_read = 0

    def connect(self) -> bool:
        raise NotImplementedError(
            f"{type(self).__name__} must implement connect()"
        )

    def read(self, limit: int = None) -> list[dict]:
        raise NotImplementedError(
            f"{type(self).__name__} must implement read()"
        )

    def disconnect(self) -> None:
        self._connected = False
        print(f"[{self.name}] Disconnected")

    def get_stats(self) -> dict:
        return {
            "name": self.name,
            "connected": self._connected,
            "records_read": self._records_read,
        }
```

Notice `type(self).__name__` instead of hardcoding `"DataSource"`. `type(self)` returns the actual class of the object — if called on a `CSVSource` instance, it returns `CSVSource`, not `DataSource`. This gives you the actual subclass name in the error message, which is more informative.

Now a concrete subclass:

```python
class CSVSource(DataSource):
    """A data source that reads from a CSV file."""

    def __init__(self, name: str, filepath: str, delimiter: str = ","):
        super().__init__(name)
        self.filepath = filepath
        self.delimiter = delimiter
        self._data = []

    def connect(self) -> bool:
        from pathlib import Path
        if not Path(self.filepath).exists():
            raise FileNotFoundError(
                f"[{self.name}] Source file not found: {self.filepath}"
            )
        self._connected = True
        print(f"[{self.name}] Connected to {self.filepath}")
        return True

    def read(self, limit: int = None) -> list[dict]:
        if not self._connected:
            raise RuntimeError(
                f"[{self.name}] Must call connect() before read()"
            )
        import csv
        records = []
        with open(self.filepath, "r", encoding="utf-8") as f:
            reader = csv.DictReader(f, delimiter=self.delimiter)
            for i, row in enumerate(reader):
                if limit is not None and i >= limit:
                    break
                records.append(dict(row))
        self._records_read += len(records)
        return records
```

`super().__init__(name)` is the critical line. Let us examine exactly what this does.

`super()` returns a proxy object that represents the parent class in the context of the current object's class hierarchy. Calling `super().__init__(name)` calls `DataSource.__init__` with `self` (the current instance) as the implicit first argument.

Without this call, `DataSource.__init__` never runs. The attributes it creates — `self._connected`, `self._records_read` — do not exist on the object. The first time `disconnect()` or `get_stats()` tries to access them, you get an `AttributeError`. This is a common mistake in Python inheritance.

The rule: **if your parent class has an `__init__`, always call `super().__init__()` as the first line of your child class's `__init__`, unless you have a very specific reason not to.**

Now the `CSVSource.__init__` adds its own attributes after the parent's:

```python
self.filepath = filepath
self.delimiter = delimiter
self._data = []
```

These are in addition to the attributes created by `DataSource.__init__`. After construction, a `CSVSource` instance has all six attributes: `name`, `_connected`, `_records_read` (from parent), plus `filepath`, `delimiter`, `_data` (from child).

---

### Method resolution order — how Python finds methods in the inheritance chain

When you call `source.read()` on a `CSVSource` instance, Python needs to find the `read` method. It follows the Method Resolution Order (MRO) — a specific, deterministic sequence of classes to check.

For a simple single-inheritance chain:

```python
class A:
    def method(self):
        print("A")

class B(A):
    def method(self):
        print("B")
        super().method()

class C(B):
    pass

c = C()
c.method()
# prints "B" then "A"
```

Python checks `C` first — no `method`. Checks `B` — finds `method`, calls it. Inside `B.method`, `super().method()` follows the MRO from `B`'s position — checks `A`, finds `method`, calls it.

`C` inherits `method` from `B` because it does not define its own. You can see the MRO explicitly:

```python
print(C.__mro__)
# (<class 'C'>, <class 'B'>, <class 'A'>, <class 'object'>)
```

Every class in Python ultimately inherits from `object`. It is the root of the class hierarchy, equivalent to Java's `Object`.

Python supports multiple inheritance — a class can inherit from more than one parent. We will not go deep into this now because it adds complexity and is rarely the right design in data engineering. The MRO becomes important in multiple inheritance scenarios; for single inheritance it is straightforward.

---

### Abstract base classes — formal contracts

Using `raise NotImplementedError` in a base class method is an informal way to say "subclasses must implement this." It only fails when the method is actually called, not when the subclass is instantiated. This allows creating incomplete subclass objects that crash later.

Python's `abc` module provides formal enforcement:

```python
from abc import ABC, abstractmethod

class DataSource(ABC):
    """Abstract base class — cannot be instantiated directly."""

    def __init__(self, name: str):
        self.name = name
        self._connected = False

    @abstractmethod
    def connect(self) -> bool:
        """
        Establish connection to the data source.
        Must be implemented by every concrete subclass.
        """
        pass

    @abstractmethod
    def read(self, limit: int = None) -> list[dict]:
        """
        Read records from the source.
        Must be implemented by every concrete subclass.
        """
        pass

    def disconnect(self) -> None:
        """Non-abstract -- shared implementation for all subclasses."""
        self._connected = False
        print(f"Disconnected from {self.name}")
```

`ABC` is a helper class that sets the class's metaclass to `ABCMeta`. This metaclass intercepts object creation and checks whether all abstract methods have been implemented. If any are missing, instantiation raises `TypeError` immediately — not later when the method is called.

```python
# Cannot instantiate -- has abstract methods
try:
    source = DataSource("test")
except TypeError as e:
    print(e)
# TypeError: Can't instantiate abstract class DataSource
# with abstract methods connect, read

# Cannot instantiate -- missing read()
class IncompleteSource(DataSource):
    def connect(self) -> bool:
        return True

try:
    source = IncompleteSource("test")
except TypeError as e:
    print(e)
# TypeError: Can't instantiate abstract class IncompleteSource
# with abstract method read
```

The abstract method body with `pass` is intentional. Abstract methods can have implementations in Python — the subclass can call `super().connect()` to access the parent's implementation while still being required to override it. This is different from Java's `abstract` keyword which explicitly prohibits a method body.

In data engineering, abstract base classes define the contracts that all connectors, writers, validators, and pipeline components must satisfy. Airflow's `BaseOperator` is an abstract base class. Every custom operator you write must implement its abstract methods. Understanding ABCs makes every framework's design immediately readable.

---

### Special methods — how Python's built-in operations work

Python's built-in operations like `len()`, `print()`, `in`, `==`, and `[]` are not magic. They call specific methods on your objects. By implementing these methods, you make your objects work naturally with Python's built-in syntax.

The full set is called the data model. We will examine the ones you will actually use in data engineering.

**`__repr__` and `__str__`**

Every object has these. They control how the object is represented as text.

`__repr__` is meant to be unambiguous and developer-facing. It should ideally return a string that could recreate the object. Convention: `ClassName(param1=value1, param2=value2)`.

`__str__` is meant to be readable and user-facing. Used by `print()` and `str()`.

When you call `print(obj)`, Python calls `str(obj)`, which calls `obj.__str__()`. If `__str__` is not defined, Python falls back to `__repr__`. If `__repr__` is not defined either, Python uses the default: `<ClassName object at 0x7f...>`, which is useless.

```python
class ColumnSchema:
    def __init__(self, name: str, data_type: str, nullable: bool = True):
        self.name = name
        self.data_type = data_type
        self.nullable = nullable

    def __repr__(self) -> str:
        return (
            f"ColumnSchema("
            f"name={self.name!r}, "
            f"data_type={self.data_type!r}, "
            f"nullable={self.nullable!r})"
        )

    def __str__(self) -> str:
        nullable_str = "NULL" if self.nullable else "NOT NULL"
        return f"{self.name} {self.data_type} {nullable_str}"


col = ColumnSchema("student_id", "INTEGER", nullable=False)
print(col)          # student_id INTEGER NOT NULL  (calls __str__)
print(repr(col))    # ColumnSchema(name='student_id', ...) (calls __repr__)
```

The `!r` in the f-string applies `repr()` to the value before inserting it. So string values appear with their quotes in the output: `name='student_id'` instead of `name=student_id`. This makes `__repr__` output valid Python syntax that could recreate the object.

**`__len__`**

Called by the built-in `len()` function.

```python
class TableSchema:
    def __init__(self, table_name: str):
        self.table_name = table_name
        self.columns: list[ColumnSchema] = []

    def add_column(self, column: ColumnSchema) -> None:
        self.columns.append(column)

    def __len__(self) -> int:
        return len(self.columns)


schema = TableSchema("students")
schema.add_column(ColumnSchema("student_id", "INTEGER", False))
schema.add_column(ColumnSchema("name", "VARCHAR"))
schema.add_column(ColumnSchema("gpa", "DECIMAL"))

print(len(schema))   # 3 -- calls __len__
```

**`__contains__`**

Called by the `in` operator: `item in obj` calls `obj.__contains__(item)`.

```python
class TableSchema:
    def __contains__(self, column_name: str) -> bool:
        return any(col.name == column_name for col in self.columns)


print("student_id" in schema)   # True
print("age" in schema)          # False
```

**`__getitem__`**

Called by square bracket access: `obj[key]` calls `obj.__getitem__(key)`.

```python
class TableSchema:
    def __getitem__(self, column_name: str) -> ColumnSchema:
        for col in self.columns:
            if col.name == column_name:
                return col
        raise KeyError(
            f"Column '{column_name}' not found in table '{self.table_name}'"
        )


col = schema["gpa"]
print(col)   # gpa DECIMAL NULL
```

**`__eq__`**

Called by the `==` operator. Without this, `==` compares object identity (same memory address) — two objects with identical data are not equal.

```python
class ColumnSchema:
    def __eq__(self, other: object) -> bool:
        if not isinstance(other, ColumnSchema):
            return NotImplemented
        return (
            self.name == other.name
            and self.data_type == other.data_type
            and self.nullable == other.nullable
        )


col1 = ColumnSchema("gpa", "DECIMAL", True)
col2 = ColumnSchema("gpa", "DECIMAL", True)
col3 = ColumnSchema("gpa", "FLOAT", True)

print(col1 == col2)   # True -- same data
print(col1 == col3)   # False -- different data_type
print(col1 is col2)   # False -- different objects in memory
```

`return NotImplemented` (not `NotImplementedError`) is the correct response when the comparison does not make sense — comparing a `ColumnSchema` to an integer, for example. It tells Python to try the comparison the other way (`other.__eq__(self)`) before giving up and returning `False`.

**`__iter__`**

Called when an object is used in a `for` loop or any context that expects an iterable.

```python
class TableSchema:
    def __iter__(self):
        return iter(self.columns)


for column in schema:
    print(column)
```

`iter(self.columns)` returns an iterator over the list. The `for` loop calls `next()` on this iterator to get each column in sequence.

---

### Properties — controlled attribute access without getters and setters

In Java, controlled attribute access requires explicit getter and setter methods:

```java
private double gpa;

public double getGpa() { return gpa; }
public void setGpa(double gpa) {
    if (gpa < 0 || gpa > 10) throw new IllegalArgumentException(...);
    this.gpa = gpa;
}
```

Callers must write `student.getGpa()` and `student.setGpa(8.5)`. If you initially made `gpa` public and later need to add validation, you must change all callers from `student.gpa` to `student.getGpa()`.

Python's property mechanism lets you start with plain attribute access and transparently add validation later without changing any calling code.

```python
class Student:
    def __init__(self, name: str, gpa: float | None = None):
        self.name = name
        self._gpa = None        # private storage
        if gpa is not None:
            self.gpa = gpa      # goes through the setter

    @property
    def gpa(self) -> float | None:
        """The student's GPA on a 0.0-10.0 scale."""
        return self._gpa

    @gpa.setter
    def gpa(self, value: float | None) -> None:
        if value is not None:
            if not isinstance(value, (int, float)):
                raise TypeError(
                    f"GPA must be a number, got {type(value).__name__}"
                )
            if not (0.0 <= value <= 10.0):
                raise ValueError(
                    f"GPA must be between 0.0 and 10.0, got {value}"
                )
        self._gpa = float(value) if value is not None else None

    @property
    def grade_category(self) -> str:
        """Computed property with no setter -- read-only."""
        if self._gpa is None:
            return "Ungraded"
        if self._gpa >= 9.0:
            return "Outstanding"
        if self._gpa >= 7.5:
            return "Good"
        return "Needs Improvement"


student = Student("Arjun", 8.5)
print(student.gpa)               # 8.5 -- calls getter
student.gpa = 9.2                # calls setter, validates
print(student.grade_category)    # Outstanding -- calls computed property

try:
    student.gpa = 11.0
except ValueError as e:
    print(e)

try:
    student.grade_category = "Excellent"   # AttributeError -- no setter
except AttributeError as e:
    print(e)
```

The calling code (`student.gpa`, `student.gpa = 9.2`) looks identical to plain attribute access. There is no sign that a method is being called. This is the key benefit — the interface is clean, and the implementation detail (that validation is happening) is hidden.

The `_gpa` private attribute is the actual storage. The property getter returns it. The property setter validates and then assigns to it. Without the separate `_gpa`, you would have infinite recursion — the setter assigns to `self.gpa`, which calls the setter, which assigns to `self.gpa`, which calls the setter...

---

### Dataclasses — removing boilerplate

A significant portion of Python classes that hold data spend most of their code on boilerplate: `__init__` parameter assignments, `__repr__` that lists all attributes, `__eq__` that compares all attributes. The logic-to-boilerplate ratio is poor.

Python's `dataclasses` module generates all of this automatically from the class's type-annotated attributes.

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
    max_retries: Optional[int] = 3
```

This generates an `__init__` with all eight parameters in order, a `__repr__` that shows all attribute values, and an `__eq__` that compares all attributes. What would be 30 lines of boilerplate is now 9 lines of attribute declarations.

`field(default_factory=list)` is the correct way to specify a mutable default. The `default_factory` is called to create a fresh list for each new instance. Using `tags: list[str] = []` directly would trigger an error — the dataclass decorator detects mutable defaults and refuses them, forcing you to use `field`.

You can add your own methods to a dataclass — the decorator only generates the boilerplate:

```python
@dataclass
class PipelineConfig:
    name: str
    source_path: str
    output_path: str
    delimiter: str = ","
    batch_size: int = 1000
    tags: list[str] = field(default_factory=list)

    def is_valid(self) -> bool:
        return all([
            bool(self.name.strip()),
            bool(self.source_path.strip()),
            bool(self.output_path.strip()),
            self.batch_size > 0,
        ])

    def with_batch_size(self, new_size: int) -> "PipelineConfig":
        """Return a new config with a different batch size."""
        from dataclasses import replace
        return replace(self, batch_size=new_size)


config = PipelineConfig(
    name="student_etl",
    source_path="/data/students.csv",
    output_path="/output/students.parquet",
    tags=["daily", "students"],
)

print(config)
print(config.is_valid())

scaled = config.with_batch_size(5000)
print(scaled.batch_size)   # 5000
print(config.batch_size)   # 1000 -- original unchanged
```

`dataclasses.replace()` creates a copy of a dataclass instance with specific fields changed. The original is untouched. This is the correct pattern for creating modified configurations — do not mutate the original, create a modified copy.

The `@dataclass(frozen=True)` variant creates immutable instances — attempting to set any attribute after creation raises a `FrozenInstanceError`. Immutable configuration objects are a sound design choice in pipelines because a pipeline stage cannot accidentally change the configuration it received.

---

### Putting everything together — a realistic example

This is what the concepts above look like combined in a real data engineering context. Read this carefully — trace through it and make sure you understand why every line is written the way it is.

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Optional
import functools


def log_method(func):
    """Decorator that logs method entry and exit with class name."""
    @functools.wraps(func)
    def wrapper(self, *args, **kwargs):
        class_name = type(self).__name__
        print(f"[{class_name}] Starting {func.__name__}")
        result = func(self, *args, **kwargs)
        print(f"[{class_name}] Completed {func.__name__}")
        return result
    return wrapper


@dataclass
class ValidationResult:
    is_valid: bool
    errors: list[str] = field(default_factory=list)

    def add_error(self, message: str) -> None:
        self.errors.append(message)
        self.is_valid = False

    def __str__(self) -> str:
        if self.is_valid:
            return "Valid"
        return f"Invalid: {'; '.join(self.errors)}"


class SchemaValidator(ABC):
    """Abstract base class for schema validation."""

    def __init__(self, schema_name: str):
        self.schema_name = schema_name

    @abstractmethod
    def validate_record(self, record: dict) -> ValidationResult:
        pass

    def validate_batch(self, records: list[dict]) -> tuple[list[dict], list[dict]]:
        """
        Validate a batch of records.
        Returns (valid_records, invalid_records).
        """
        valid = []
        invalid = []
        for record in records:
            result = self.validate_record(record)
            if result.is_valid:
                valid.append(record)
            else:
                print(f"  Invalid record {record.get('student_id')}: {result}")
                invalid.append(record)
        return valid, invalid

    def __repr__(self) -> str:
        return f"{type(self).__name__}(schema={self.schema_name!r})"


class StudentValidator(SchemaValidator):
    """Validates student records against expected schema and business rules."""

    def __init__(self):
        super().__init__("students")
        self._valid_cities = {
            "Mumbai", "Delhi", "Chennai",
            "Bangalore", "Hyderabad", "Pune", "Ahmedabad"
        }

    def validate_record(self, record: dict) -> ValidationResult:
        result = ValidationResult(is_valid=True)

        if not isinstance(record.get("student_id"), int):
            result.add_error("student_id must be an integer")

        name = record.get("name", "")
        if not name or not name.strip():
            result.add_error("name cannot be empty")
        elif len(name.strip()) < 2:
            result.add_error("name must be at least 2 characters")

        gpa = record.get("gpa")
        if gpa is not None:
            if not isinstance(gpa, (int, float)):
                result.add_error("gpa must be a number")
            elif not (0.0 <= gpa <= 10.0):
                result.add_error(f"gpa {gpa} is outside valid range [0.0, 10.0]")

        city = record.get("city", "")
        if city and city not in self._valid_cities:
            result.add_error(
                f"city '{city}' not in known cities: {sorted(self._valid_cities)}"
            )

        return result

    @property
    def valid_cities(self) -> frozenset:
        """Read-only view of valid cities."""
        return frozenset(self._valid_cities)

    def add_valid_city(self, city: str) -> None:
        self._valid_cities.add(city.strip().title())


validator = StudentValidator()
print(repr(validator))

records = [
    {"student_id": 1, "name": "Arjun Mehta", "gpa": 8.5, "city": "Mumbai"},
    {"student_id": 2, "name": "Priya Sharma", "gpa": 11.0, "city": "Delhi"},
    {"student_id": 3, "name": "",             "gpa": 7.8, "city": "Mumbai"},
    {"student_id": 4, "name": "Sneha Iyer",   "gpa": 8.9, "city": "Kolkata"},
    {"student_id": 5, "name": "Karan Patel",  "gpa": None, "city": "Ahmedabad"},
]

print(f"\nValidating {len(records)} records:")
valid, invalid = validator.validate_batch(records)

print(f"\nValid records: {len(valid)}")
print(f"Invalid records: {len(invalid)}")

validator.add_valid_city("Kolkata")
print(f"\nAfter adding Kolkata, re-validating:")
valid2, invalid2 = validator.validate_batch(records)
print(f"Valid: {len(valid2)}, Invalid: {len(invalid2)}")
```

Run this. Read every output line and trace it back to the code that produced it. Make sure you understand why Priya's record is invalid (gpa of 11.0), why Sneha's record is invalid initially but valid after adding Kolkata, why Karan's record is valid (None gpa is allowed).

---

### Exercises — Lesson 2

These are not syntax exercises. They require you to think about design — which class, what methods, what relationships.

1. Create a class hierarchy for a pipeline monitoring system. Define an abstract base class `PipelineEvent` with attributes `pipeline_name` (str), `timestamp` (use `datetime.now()` from the `datetime` module), and an abstract method `to_log_string() -> str`. Implement three concrete subclasses: `PipelineStarted` (adds no extra attributes), `PipelineCompleted` (adds `records_processed: int` and `duration_seconds: float`), and `PipelineFailed` (adds `error_message: str` and `step_name: str`). Make all three dataclasses. Add `__str__` that delegates to `to_log_string()`.
    
2. Create a class `DataTypeRegistry` that maps Python types to SQL type strings. It should have a class-level dictionary as the registry. Implement a class method `register(python_type, sql_type: str)` and a class method `get_sql_type(python_type) -> str` that raises `KeyError` with a helpful message if the type is not registered. Add a static method `is_valid_sql_type(type_str: str) -> bool` that checks against a fixed set of valid SQL types. Pre-populate the registry for `int`, `float`, `str`, `bool`, and Python's `date` and `datetime` types.
    
3. Create a class `RecordBuffer` that holds records in memory up to a maximum size. When the buffer is full, it automatically flushes (in this exercise, just print the records and clear the buffer). Implement `__len__` (number of records), `__contains__` (check if a record with a specific student_id exists), `add(record: dict)` which adds a record and flushes if at capacity, and `flush()` which processes and clears all records. The max size should be configurable in `__init__` with a default of 3.
    

---

### Answers — Lesson 2

**Exercise 1**

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from datetime import datetime


class PipelineEvent(ABC):
    """Abstract base for all pipeline monitoring events."""

    def __init__(self, pipeline_name: str):
        self.pipeline_name = pipeline_name
        self.timestamp = datetime.now()

    @abstractmethod
    def to_log_string(self) -> str:
        pass

    def __str__(self) -> str:
        return self.to_log_string()


@dataclass
class PipelineStarted(PipelineEvent):
    pipeline_name: str

    def __post_init__(self):
        super().__init__(self.pipeline_name)

    def to_log_string(self) -> str:
        return (
            f"[{self.timestamp.isoformat()}] "
            f"STARTED pipeline '{self.pipeline_name}'"
        )


@dataclass
class PipelineCompleted(PipelineEvent):
    pipeline_name: str
    records_processed: int
    duration_seconds: float

    def __post_init__(self):
        super().__init__(self.pipeline_name)

    def to_log_string(self) -> str:
        return (
            f"[{self.timestamp.isoformat()}] "
            f"COMPLETED pipeline '{self.pipeline_name}' — "
            f"{self.records_processed} records in {self.duration_seconds:.2f}s"
        )


@dataclass
class PipelineFailed(PipelineEvent):
    pipeline_name: str
    error_message: str
    step_name: str

    def __post_init__(self):
        super().__init__(self.pipeline_name)

    def to_log_string(self) -> str:
        return (
            f"[{self.timestamp.isoformat()}] "
            f"FAILED pipeline '{self.pipeline_name}' "
            f"at step '{self.step_name}' — {self.error_message}"
        )


events = [
    PipelineStarted("student_etl"),
    PipelineCompleted("student_etl", records_processed=1500, duration_seconds=4.7),
    PipelineFailed("inventory_etl", error_message="Connection refused", step_name="extract"),
]

for event in events:
    print(event)
```

Note on `__post_init__`: when using `@dataclass`, the generated `__init__` calls `__post_init__` after setting all attributes. This is how you call the parent's `__init__` when mixing dataclasses with inheritance — you cannot call `super().__init__()` at the top of a dataclass `__init__` because the dataclass generates its own `__init__`.

**Exercise 2**

```python
from datetime import date, datetime


class DataTypeRegistry:
    _registry: dict = {}
    _valid_sql_types = {
        "INTEGER", "BIGINT", "SMALLINT",
        "FLOAT", "DOUBLE", "DECIMAL",
        "VARCHAR", "TEXT", "CHAR",
        "BOOLEAN",
        "DATE", "TIMESTAMP", "TIME",
    }

    @classmethod
    def register(cls, python_type: type, sql_type: str) -> None:
        if not cls.is_valid_sql_type(sql_type):
            raise ValueError(
                f"'{sql_type}' is not a valid SQL type. "
                f"Valid types: {sorted(cls._valid_sql_types)}"
            )
        cls._registry[python_type] = sql_type
        print(f"Registered: {python_type.__name__} -> {sql_type}")

    @classmethod
    def get_sql_type(cls, python_type: type) -> str:
        if python_type not in cls._registry:
            registered = [t.__name__ for t in cls._registry]
            raise KeyError(
                f"No SQL type registered for Python type '{python_type.__name__}'. "
                f"Registered types: {registered}"
            )
        return cls._registry[python_type]

    @staticmethod
    def is_valid_sql_type(type_str: str) -> bool:
        base_type = type_str.upper().split("(")[0].strip()
        return base_type in DataTypeRegistry._valid_sql_types


DataTypeRegistry.register(int,      "INTEGER")
DataTypeRegistry.register(float,    "FLOAT")
DataTypeRegistry.register(str,      "VARCHAR")
DataTypeRegistry.register(bool,     "BOOLEAN")
DataTypeRegistry.register(date,     "DATE")
DataTypeRegistry.register(datetime, "TIMESTAMP")

print(DataTypeRegistry.get_sql_type(int))
print(DataTypeRegistry.get_sql_type(datetime))

try:
    DataTypeRegistry.get_sql_type(list)
except KeyError as e:
    print(e)

print(DataTypeRegistry.is_valid_sql_type("VARCHAR(100)"))
print(DataTypeRegistry.is_valid_sql_type("BLOB"))
```

**Exercise 3**

```python
class RecordBuffer:
    """
    In-memory buffer for records with automatic flushing at capacity.
    """

    def __init__(self, max_size: int = 3):
        self._max_size = max_size
        self._records: list[dict] = []
        self._total_flushed = 0

    def add(self, record: dict) -> None:
        self._records.append(record)
        print(f"  Added record {record.get('student_id')} "
              f"(buffer: {len(self)}/{self._max_size})")
        if len(self) >= self._max_size:
            self.flush()

    def flush(self) -> int:
        if not self._records:
            print("  Buffer is empty, nothing to flush")
            return 0
        count = len(self._records)
        print(f"  Flushing {count} records:")
        for record in self._records:
            print(f"    -> {record}")
        self._total_flushed += count
        self._records = []
        print(f"  Buffer cleared. Total flushed so far: {self._total_flushed}")
        return count

    def __len__(self) -> int:
        return len(self._records)

    def __contains__(self, student_id: int) -> bool:
        return any(r.get("student_id") == student_id for r in self._records)

    def __repr__(self) -> str:
        return (
            f"RecordBuffer("
            f"size={len(self)}/{self._max_size}, "
            f"total_flushed={self._total_flushed})"
        )


buffer = RecordBuffer(max_size=3)

records = [
    {"student_id": 1, "name": "Arjun", "gpa": 8.5},
    {"student_id": 2, "name": "Priya", "gpa": 9.1},
    {"student_id": 3, "name": "Rohan", "gpa": 7.8},  # flush triggers here
    {"student_id": 4, "name": "Sneha", "gpa": 8.9},
    {"student_id": 5, "name": "Karan", "gpa": 6.5},
]

for record in records:
    buffer.add(record)

print(f"\nBuffer after adding all records: {buffer}")
print(f"Student 4 in buffer: {4 in buffer}")
print(f"Student 1 in buffer: {1 in buffer}")

print("\nFinal flush:")
buffer.flush()
print(repr(buffer))
```

Lesson 2 is complete. Tell me when you are ready for Lesson 3.