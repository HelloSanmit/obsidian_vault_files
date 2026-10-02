
## Phase 2, Lesson 2 — Object Oriented Programming in Python

---

### Before anything else — Python syntax that is new to you

You came from Java and C++. Python has several syntax elements that do not exist in those languages. I will explain each one the first time it appears rather than assuming you know it. The first one you asked about is `->` and type hints in general. Let me cover all the new syntax upfront so nothing surprises you mid-lesson.

---

### Type hints — Python's way of documenting types

In Java, you declare types. The compiler enforces them.

```java
// Java
int add(int a, int b) {
    return a + b;
}
```

The return type `int` and parameter types `int` are mandatory. The compiler rejects code that violates them.

In Python, variables and parameters have no declared type by default. Python figures out types at runtime.

```python
# Python without type hints
def add(a, b):
    return a + b
```

This works. But it tells you nothing about what `a` and `b` should be. Are they integers? Floats? Strings? You have to read the function body or its documentation to find out.

Type hints were added to Python in version 3.5 to solve this problem. They let you annotate parameters and return values with expected types.

```python
# Python with type hints
def add(a: int, b: int) -> int:
    return a + b
```

The `: int` after each parameter name is the type hint for that parameter. The `-> int` before the colon that opens the function body is the return type hint.

**The critical difference from Java: Python does not enforce type hints at runtime.** You can write `add("hello", "world")` and Python will not complain — it will return `"helloworld"` because string concatenation with `+` works. The type hints are documentation and tooling assistance, not enforcement.

What they do give you:

VS Code reads type hints and gives you autocomplete based on the declared type. If you write `result = add(2, 3)` and then type `result.`, VS Code knows result is an int and suggests integer methods.

VS Code also underlines mismatches with a warning. If you write `add(2.5, 3)` it will warn you that 2.5 is a float, not an int.

In production data engineering teams, a tool called `mypy` runs as part of the automated testing process and actually enforces type hints — but this is a separate step, not Python itself.

The `->` specifically means "this function returns." `-> int` means "this function returns an integer." `-> None` means the function returns nothing. `-> list[dict]` means it returns a list of dictionaries.

More type hint syntax you will see:

```python
from typing import Optional, Union

def get_gpa(student_id: int) -> Optional[float]:
    # Optional[float] means the return is either a float or None
    pass

def process(value: Union[int, str]) -> str:
    # Union[int, str] means the parameter can be int or str
    pass

# Python 3.10+ shorthand for Optional and Union
def get_gpa(student_id: int) -> float | None:
    pass

def process(value: int | str) -> str:
    pass
```

`Optional[float]` is exactly equivalent to `float | None`. It means the value can be a float or it can be None. This is extremely common in data engineering because real data has missing values — a student may or may not have a GPA.

Now you understand all the type hint syntax. Every function I write from this point will have type hints. Read them as documentation — they tell you exactly what a function expects and returns.

---

### f-strings — Python's string formatting

You will see `f"..."` throughout the code. This is an f-string — Python's modern string formatting syntax introduced in Python 3.6.

```python
name = "Arjun"
gpa = 8.5

# Old way -- do not use this
message = "Student %s has GPA %f" % (name, gpa)

# Also old -- do not use this
message = "Student {} has GPA {}".format(name, gpa)

# Modern way -- use this always
message = f"Student {name} has GPA {gpa}"
print(message)  # Student Arjun has GPA 8.5
```

The `f` prefix before the opening quote makes it an f-string. Anything inside `{}` is evaluated as a Python expression and inserted into the string.

```python
# Expressions inside f-strings
print(f"GPA doubled: {gpa * 2}")
print(f"Name uppercase: {name.upper()}")
print(f"Is high achiever: {gpa > 8.0}")

# Formatting numbers
print(f"GPA: {gpa:.2f}")    # 2 decimal places: 8.50
print(f"GPA: {gpa:.1f}")    # 1 decimal place: 8.5
print(f"Count: {1000000:,}")  # thousands separator: 1,000,000
```

The `:` inside the curly braces specifies formatting. `:.2f` means float with 2 decimal places. `:,` means add a thousands separator.

You will use f-strings everywhere — log messages, error messages, file paths, SQL query strings. Use them by default over the older formats.

---

### List comprehensions — Python's compact loop syntax

You will see this syntax constantly in Python code:

```python
numbers = [1, 2, 3, 4, 5]

# Traditional loop
doubled = []
for n in numbers:
    doubled.append(n * 2)

# List comprehension -- identical result
doubled = [n * 2 for n in numbers]

# With a condition
even_doubled = [n * 2 for n in numbers if n % 2 == 0]
```

`[expression for item in iterable if condition]` is a list comprehension. It creates a new list by applying an expression to each item in an iterable, optionally filtering with a condition. The result is identical to writing the loop and append manually — it is just more concise.

In data engineering, list comprehensions appear everywhere for transforming rows of data:

```python
rows = [
    {"name": "  Arjun ", "gpa": "8.5"},
    {"name": "Priya  ", "gpa": "9.1"},
]

cleaned = [
    {"name": row["name"].strip(), "gpa": float(row["gpa"])}
    for row in rows
]
```

Dictionary comprehensions work the same way:

```python
students = [("Arjun", 8.5), ("Priya", 9.1), ("Rohan", 7.8)]

# Create a dict mapping name to GPA
gpa_lookup = {name: gpa for name, gpa in students}
# {"Arjun": 8.5, "Priya": 9.1, "Rohan": 7.8}
```

Now you have the Python syntax you need. Into OOP.

---

### What a class is — building the mental model from Java

You already know what a class is from Java. A class is a blueprint. An object is an instance of that blueprint. The class defines what data an object holds (attributes) and what it can do (methods). Every object created from the same class has the same structure but its own independent data.

Python classes are conceptually identical. The syntax is different and some mechanisms work differently. Let us build a class from scratch and examine every part.

Create a new file in your `de_learning` folder called `lesson2.py`.

```python
class Student:
    
    def __init__(self, student_id: int, name: str, gpa: float | None):
        self.student_id = student_id
        self.name = name
        self.gpa = gpa

arjun = Student(1, "Arjun Mehta", 8.5)
priya = Student(2, "Priya Sharma", 9.1)

print(arjun.name)       # Arjun Mehta
print(priya.gpa)        # 9.1
print(arjun.student_id) # 1
```

Run this. Make sure it works before reading further.

Now let us examine every part of this.

---

### `__init__` — the constructor

`__init__` is Python's constructor. It runs automatically when you write `Student(1, "Arjun Mehta", 8.5)`.

In Java your constructor looks like this:

```java
public Student(int studentId, String name, double gpa) {
    this.studentId = studentId;
    this.name = name;
    this.gpa = gpa;
}
```

In Python:

```python
def __init__(self, student_id: int, name: str, gpa: float | None):
    self.student_id = student_id
    self.name = name
    self.gpa = gpa
```

Three differences from Java.

First, the constructor is named `__init__` — always, for every class, without exception. In Java the constructor has the same name as the class. In Python it is always `__init__`.

Second, `self` is the first parameter. `self` is Python's equivalent of Java's `this`. It is a reference to the object being created. Unlike Java where `this` is implicit, in Python you must explicitly write `self` as the first parameter of every instance method. Python passes it automatically when you call the method — you never write `student.__init__(self, ...)` yourself, just `Student(...)`.

Third, you create instance attributes by assigning to `self.attribute_name` inside `__init__`. In Java you declare instance variables at the class level first:

```java
// Java -- declare first
private int studentId;
private String name;

// Then assign in constructor
this.studentId = studentId;
```

In Python there is no separate declaration. The first assignment to `self.student_id` inside `__init__` both declares and initializes the attribute. The attribute does not exist before `__init__` runs.

---

### `self` — understanding it deeply

`self` confuses many beginners. Let us be precise.

When you write:

```python
arjun = Student(1, "Arjun Mehta", 8.5)
```

Python internally does something equivalent to:

```python
arjun = Student.__new__(Student)   # create empty object
Student.__init__(arjun, 1, "Arjun Mehta", 8.5)  # initialize it
```

The object `arjun` is passed as the first argument to `__init__`. Inside `__init__`, that argument is named `self`. So `self` is literally the object being created or the object the method is being called on.

When you write:

```python
arjun.get_grade()
```

Python internally translates this to:

```python
Student.get_grade(arjun)
```

`arjun` gets passed as `self`. Inside `get_grade`, `self` refers to `arjun`. Every instance method works this way.

The name `self` is a convention, not a keyword. You could name it anything. But everyone uses `self` and you should too — deviating from it makes your code unreadable to every other Python developer.

---

### Instance attributes vs class attributes

```python
class Student:
    
    school_name = "Delhi Technical University"  # class attribute
    student_count = 0                           # class attribute

    def __init__(self, student_id: int, name: str, gpa: float | None):
        Student.student_count += 1              # modify class attribute
        self.student_id = student_id            # instance attribute
        self.name = name                        # instance attribute
        self.gpa = gpa                          # instance attribute


arjun = Student(1, "Arjun Mehta", 8.5)
priya = Student(2, "Priya Sharma", 9.1)

print(Student.school_name)    # Delhi Technical University
print(arjun.school_name)      # Delhi Technical University -- accessible via instance too
print(Student.student_count)  # 2
print(arjun.student_count)    # 2 -- same value, shared

print(arjun.name)   # Arjun Mehta
print(priya.name)   # Priya Sharma -- different per instance
```

Class attributes are defined directly inside the class body, outside any method. They are shared across all instances — one copy exists at the class level. Every instance can read them. When you modify a class attribute through the class name (`Student.student_count += 1`), every instance sees the updated value.

Instance attributes are defined inside `__init__` with `self.`. Each instance has its own copy. `arjun.name` and `priya.name` are completely independent.

An important subtlety: if you assign to an attribute through an instance (`arjun.school_name = "IIT"`), Python creates a new instance attribute that shadows the class attribute for that specific instance. The class attribute is unchanged. Other instances still see the original. This behavior surprises people — be aware of it.

In data engineering, class attributes are used for:

- Default configuration values shared across all instances
- Counters tracking how many connections have been opened
- Registries of all registered pipeline types or data source types

---

### Methods — the full picture

A method is a function defined inside a class. There are three kinds.

**Instance methods** — the most common kind. They receive `self` as the first parameter and operate on the specific object they are called on.

```python
class Student:
    def __init__(self, student_id: int, name: str, gpa: float | None):
        self.student_id = student_id
        self.name = name
        self.gpa = gpa

    def is_high_achiever(self) -> bool:
        """Return True if GPA is 9.0 or above."""
        if self.gpa is None:
            return False
        return self.gpa >= 9.0

    def describe(self) -> str:
        """Return a human-readable description of this student."""
        gpa_str = f"{self.gpa:.2f}" if self.gpa is not None else "N/A"
        return f"Student {self.student_id}: {self.name} (GPA: {gpa_str})"


arjun = Student(1, "Arjun Mehta", 8.5)
priya = Student(2, "Priya Sharma", 9.1)

print(arjun.describe())
print(priya.is_high_achiever())  # True
print(arjun.is_high_achiever())  # False
```

**Class methods** — they receive the class itself as the first argument instead of an instance. Decorated with `@classmethod`. Used to create alternative constructors.

```python
class Student:
    def __init__(self, student_id: int, name: str, gpa: float | None):
        self.student_id = student_id
        self.name = name
        self.gpa = gpa

    @classmethod
    def from_dict(cls, data: dict) -> "Student":
        """Create a Student from a dictionary."""
        return cls(
            student_id=int(data["student_id"]),
            name=data["name"],
            gpa=float(data["gpa"]) if data.get("gpa") else None
        )

    @classmethod
    def from_csv_row(cls, row: str) -> "Student":
        """Create a Student from a comma-separated string."""
        parts = row.strip().split(",")
        return cls(
            student_id=int(parts[0]),
            name=parts[1].strip(),
            gpa=float(parts[2]) if parts[2].strip() else None
        )


# Two ways to create a Student
s1 = Student(1, "Arjun Mehta", 8.5)
s2 = Student.from_dict({"student_id": "2", "name": "Priya Sharma", "gpa": "9.1"})
s3 = Student.from_csv_row("3, Rohan Verma, 7.8")

print(s2.name)  # Priya Sharma
print(s3.gpa)   # 7.8
```

`cls` is the convention for the class method's first parameter, equivalent to `self` for instance methods. `cls` refers to the class itself, not an instance. Calling `cls(...)` is the same as calling `Student(...)` — it creates a new instance.

The `-> "Student"` in quotes is a forward reference. Inside the class definition, the name `Student` is not fully defined yet, so you quote it. Python 3.10+ handles this automatically, but quoting is the safe convention for earlier versions.

`from_dict` and `from_csv_row` are alternative constructors — different ways to create a Student object from different data sources. This pattern appears constantly in data engineering. You have one primary constructor and multiple class methods that handle different input formats, all ultimately calling the primary constructor.

**Static methods** — they receive neither `self` nor `cls`. They are regular functions that live inside the class namespace for organizational purposes.

```python
class Student:
    def __init__(self, student_id: int, name: str, gpa: float | None):
        self.student_id = student_id
        self.name = name
        self.gpa = gpa

    @staticmethod
    def validate_gpa(gpa: float) -> bool:
        """Check if a GPA value is within the valid range of 0.0 to 10.0."""
        return 0.0 <= gpa <= 10.0

    @staticmethod
    def normalize_name(name: str) -> str:
        """Standardize a name to title case with no extra whitespace."""
        return " ".join(name.split()).title()


print(Student.validate_gpa(8.5))   # True
print(Student.validate_gpa(11.0))  # False
print(Student.normalize_name("  arjun   mehta  "))  # Arjun Mehta
```

Static methods cannot access class or instance state. They could technically be standalone functions — they live in the class purely because they are logically related to it. Use them sparingly. If a method does not need `self` or `cls`, ask whether it truly belongs in the class or whether it should be a standalone function.

---

### Privacy conventions — single underscore and double underscore

Python does not have true private members like Java. Instead it uses naming conventions.

**Single underscore prefix `_name`** — means "this is intended for internal use." It is a signal to other developers: do not access this from outside the class unless you really know what you are doing. Python does not enforce this — you can still access `obj._attribute` from outside. It is purely a social contract.

**Double underscore prefix `__name`** — triggers name mangling. Python renames the attribute to `_ClassName__attribute` internally. This makes it harder (but not impossible) to access from outside and prevents accidental override in subclasses.

```python
class DataPipeline:
    def __init__(self, name: str):
        self.name = name              # public -- access freely
        self._run_count = 0          # protected -- internal use intended
        self.__secret_key = "abc123" # private -- name mangled

    def get_run_count(self) -> int:
        return self._run_count

pipeline = DataPipeline("sales")

print(pipeline.name)          # works fine
print(pipeline._run_count)    # works but you are violating the convention
print(pipeline.__secret_key)  # AttributeError -- name mangled
print(pipeline._DataPipeline__secret_key)  # works -- mangled name
```

In practice, use single underscore for internal attributes and methods. Use double underscore only when you specifically need to prevent subclass name collision. Most Python codebases use single underscore far more than double underscore.

---

### Inheritance — reusing and extending behavior

Inheritance in Python works identically to Java in concept. A child class inherits all attributes and methods of the parent class and can override or extend them.

```python
class DataSource:
    """Base class representing any data source a pipeline can read from."""

    def __init__(self, name: str, description: str = ""):
        self.name = name
        self.description = description
        self._connected = False

    def connect(self) -> bool:
        """Establish connection to the data source. Override in subclass."""
        raise NotImplementedError(
            f"{self.__class__.__name__} must implement connect()"
        )

    def read(self, batch_size: int = 1000) -> list[dict]:
        """Read data from the source. Override in subclass."""
        raise NotImplementedError(
            f"{self.__class__.__name__} must implement read()"
        )

    def disconnect(self) -> None:
        """Close the connection."""
        self._connected = False
        print(f"Disconnected from {self.name}")

    def describe(self) -> str:
        """Return a description of this data source."""
        status = "connected" if self._connected else "disconnected"
        return f"[{status}] {self.name}: {self.description}"
```

Now create two subclasses:

```python
class CSVSource(DataSource):
    """A data source that reads from a CSV file."""

    def __init__(self, name: str, filepath: str, delimiter: str = ","):
        super().__init__(name, description=f"CSV file at {filepath}")
        self.filepath = filepath
        self.delimiter = delimiter

    def connect(self) -> bool:
        print(f"Checking that {self.filepath} exists and is readable")
        self._connected = True
        return True

    def read(self, batch_size: int = 1000) -> list[dict]:
        if not self._connected:
            raise RuntimeError("Must call connect() before read()")
        print(f"Reading up to {batch_size} rows from {self.filepath}")
        return []  # real implementation would read the file


class DatabaseSource(DataSource):
    """A data source that reads from a PostgreSQL database."""

    def __init__(self, name: str, host: str, database: str, table: str):
        super().__init__(name, description=f"Table {table} in {database} on {host}")
        self.host = host
        self.database = database
        self.table = table

    def connect(self) -> bool:
        print(f"Connecting to {self.database} at {self.host}")
        self._connected = True
        return True

    def read(self, batch_size: int = 1000) -> list[dict]:
        if not self._connected:
            raise RuntimeError("Must call connect() before read()")
        print(f"SELECT * FROM {self.table} LIMIT {batch_size}")
        return []

    def read_with_filter(self, where_clause: str, batch_size: int = 1000) -> list[dict]:
        """Extended method that does not exist in the base class."""
        if not self._connected:
            raise RuntimeError("Must call connect() before read()")
        print(f"SELECT * FROM {self.table} WHERE {where_clause} LIMIT {batch_size}")
        return []
```

Notice `super().__init__(name, description=...)`. This calls the parent class constructor. In Java you write `super(name, description)`. In Python it is `super().__init__(name, description)`. The `super()` call is important — without it, the parent's `__init__` never runs and attributes defined there (`self.name`, `self._connected`) are never created. Your child class would fail when trying to access `self.name`.

Now use them together:

```python
csv = CSVSource("sales_data", "sales.csv", delimiter=";")
db = DatabaseSource("customers", "localhost", "production", "customers")

sources: list[DataSource] = [csv, db]

for source in sources:
    source.connect()
    print(source.describe())
    data = source.read(batch_size=500)
    source.disconnect()
    print()
```

`list[DataSource]` as the type hint on `sources` means: a list where every element is a DataSource or any subclass of it. Both CSVSource and DatabaseSource qualify. The `for` loop calls `connect()`, `describe()`, `read()`, and `disconnect()` on each — and the correct version is called based on the actual type of each object. This is polymorphism.

`source.read_with_filter(...)` would fail here because `DataSource` does not have that method — only `DatabaseSource` does. You would need to either check the type first or hold the reference as `DatabaseSource` type. This is where Python's duck typing philosophy matters — we will discuss it after abstract classes.

---

### Abstract base classes — enforcing the contract

`raise NotImplementedError` in the base class is an informal way to say "subclasses must implement this." Python has a formal mechanism that enforces it at instantiation time.

```python
from abc import ABC, abstractmethod

class DataSource(ABC):
    """Abstract base class for all data sources."""

    def __init__(self, name: str):
        self.name = name
        self._connected = False

    @abstractmethod
    def connect(self) -> bool:
        """Establish connection to the data source."""
        pass

    @abstractmethod
    def read(self, batch_size: int = 1000) -> list[dict]:
        """Read data from the source."""
        pass

    def disconnect(self) -> None:
        self._connected = False
        print(f"Disconnected from {self.name}")
```

`ABC` stands for Abstract Base Class. Inheriting from it makes your class abstract. `@abstractmethod` marks a method as abstract — subclasses must implement it.

What happens if you try to instantiate DataSource directly:

```python
source = DataSource("test")
# TypeError: Can't instantiate abstract class DataSource
# with abstract methods connect, read
```

What happens if a subclass does not implement all abstract methods:

```python
class IncompleteSource(DataSource):
    def connect(self) -> bool:
        return True
    # forgot to implement read()

s = IncompleteSource("test")
# TypeError: Can't instantiate abstract class IncompleteSource
# with abstract method read
```

Python catches both mistakes at the moment you try to create an object, not at the moment you call the missing method. This is the difference from `raise NotImplementedError` which only fails when the method is actually called.

In data engineering, abstract base classes define contracts. Every data connector must implement `connect` and `read`. Every data writer must implement `write` and `close`. The abstract class enforces that contract so a broken implementation cannot accidentally be used in a pipeline.

---

### Special methods — making your objects behave like Python built-ins

Python defines a set of special methods with double underscores on both sides. These are called dunder methods (double underscore). They let your objects integrate with Python's built-in operations.

You already know `__init__`. Here are the others you will use in data engineering.

```python
class DataSchema:
    """Represents the schema of a dataset — column names and their types."""

    def __init__(self, name: str, columns: dict[str, str]):
        self.name = name
        self.columns = columns

    def __repr__(self) -> str:
        """Called by repr() and in the Python interpreter."""
        return f"DataSchema(name={self.name!r}, columns={self.columns!r})"

    def __str__(self) -> str:
        """Called by print() and str()."""
        lines = [f"Schema: {self.name}"]
        for col_name, col_type in self.columns.items():
            lines.append(f"  {col_name}: {col_type}")
        return "\n".join(lines)

    def __len__(self) -> int:
        """Called by len()."""
        return len(self.columns)

    def __contains__(self, column_name: str) -> bool:
        """Called by the 'in' operator."""
        return column_name in self.columns

    def __getitem__(self, column_name: str) -> str:
        """Called by square bracket access: schema['name']."""
        if column_name not in self.columns:
            raise KeyError(f"Column '{column_name}' not in schema '{self.name}'")
        return self.columns[column_name]

    def __eq__(self, other: object) -> bool:
        """Called by the == operator."""
        if not isinstance(other, DataSchema):
            return NotImplemented
        return self.name == other.name and self.columns == other.columns

    def __iter__(self):
        """Called by for loops and anything that iterates."""
        return iter(self.columns)


schema = DataSchema("students", {
    "student_id": "INTEGER",
    "name":       "VARCHAR(100)",
    "gpa":        "DECIMAL(3,2)",
    "city":       "VARCHAR(50)",
})

print(schema)                     # calls __str__
print(repr(schema))               # calls __repr__
print(len(schema))                # calls __len__ -- 4
print("gpa" in schema)            # calls __contains__ -- True
print("age" in schema)            # calls __contains__ -- False
print(schema["gpa"])              # calls __getitem__ -- DECIMAL(3,2)

for column in schema:             # calls __iter__
    print(column)

schema2 = DataSchema("students", {
    "student_id": "INTEGER",
    "name":       "VARCHAR(100)",
    "gpa":        "DECIMAL(3,2)",
    "city":       "VARCHAR(50)",
})
print(schema == schema2)          # calls __eq__ -- True
```

Run this. Observe that `print(schema)` calls `__str__` and gives the formatted output. `repr(schema)` calls `__repr__` and gives the technical representation.

The `!r` inside the f-string in `__repr__` (`{self.name!r}`) applies `repr()` to the value before inserting it. So string values appear with their quotes: `name='students'` instead of `name=students`.

`NotImplemented` (not to be confused with `NotImplementedError`) in `__eq__` is a special Python singleton returned when a comparison does not make sense between two types. It tells Python to try the comparison the other way around.

---

### Dataclasses — eliminating boilerplate

Writing `__init__`, `__repr__`, and `__eq__` manually for every class that mainly holds data is repetitive. Python's `dataclass` decorator generates them automatically.

```python
from dataclasses import dataclass, field
from typing import Optional

@dataclass
class ColumnDefinition:
    """Represents a single column in a database table."""
    name: str
    data_type: str
    nullable: bool = True
    primary_key: bool = False
    default_value: Optional[str] = None

    def is_required(self) -> bool:
        """A column is required if it is not nullable and has no default."""
        return not self.nullable and self.default_value is None


@dataclass
class TableDefinition:
    """Represents a complete database table schema."""
    table_name: str
    columns: list[ColumnDefinition] = field(default_factory=list)

    def add_column(self, column: ColumnDefinition) -> None:
        self.columns.append(column)

    def get_primary_keys(self) -> list[str]:
        return [col.name for col in self.columns if col.primary_key]

    def get_required_columns(self) -> list[str]:
        return [col.name for col in self.columns if col.is_required()]

    def __len__(self) -> int:
        return len(self.columns)


students_table = TableDefinition(table_name="students")
students_table.add_column(
    ColumnDefinition("student_id", "INTEGER", nullable=False, primary_key=True)
)
students_table.add_column(
    ColumnDefinition("name", "VARCHAR(100)", nullable=False)
)
students_table.add_column(
    ColumnDefinition("gpa", "DECIMAL(3,2)", nullable=True)
)
students_table.add_column(
    ColumnDefinition("city", "VARCHAR(50)", nullable=True)
)

print(students_table)
print(f"Columns: {len(students_table)}")
print(f"Primary keys: {students_table.get_primary_keys()}")
print(f"Required columns: {students_table.get_required_columns()}")

col1 = ColumnDefinition("student_id", "INTEGER", nullable=False, primary_key=True)
col2 = ColumnDefinition("student_id", "INTEGER", nullable=False, primary_key=True)
print(col1 == col2)  # True -- dataclass generates __eq__ automatically
```

The `@dataclass` decorator reads the class body's type-annotated attributes and automatically generates:

- `__init__` with all attributes as parameters in order
- `__repr__` that shows the class name and all attribute values
- `__eq__` that compares all attributes

Attributes without defaults must come before attributes with defaults — same rule as function parameters.

`field(default_factory=list)` is the correct way to set a mutable default. As explained in Lesson 1, using `columns: list[ColumnDefinition] = []` directly would share one list object across all instances. `field(default_factory=list)` creates a fresh list for each instance.

You can still add your own methods to a dataclass — `add_column`, `get_primary_keys`, `get_required_columns`, and `__len__` above are custom methods added to a dataclass. The decorator generates the boilerplate; you add the logic.

---

### Properties — controlled attribute access

In Java you write getters and setters to control attribute access. Python has a cleaner mechanism called properties.

```python
class Student:
    def __init__(self, student_id: int, name: str, gpa: float | None):
        self.student_id = student_id
        self.name = name
        self._gpa = gpa  # stored in private attribute

    @property
    def gpa(self) -> float | None:
        """Get the GPA value."""
        return self._gpa

    @gpa.setter
    def gpa(self, value: float | None) -> None:
        """Set the GPA, validating the range."""
        if value is not None and not (0.0 <= value <= 10.0):
            raise ValueError(
                f"GPA must be between 0.0 and 10.0, got {value}"
            )
        self._gpa = value

    @property
    def grade_category(self) -> str:
        """Computed property -- has no setter."""
        if self._gpa is None:
            return "Unknown"
        if self._gpa >= 9.0:
            return "Outstanding"
        if self._gpa >= 7.5:
            return "Good"
        return "Needs Improvement"


student = Student(1, "Arjun", 8.5)
print(student.gpa)              # 8.5 -- calls the getter
student.gpa = 9.0               # calls the setter
print(student.grade_category)   # Outstanding -- computed on the fly

try:
    student.gpa = 11.0          # raises ValueError
except ValueError as e:
    print(e)
```

`@property` turns a method into an attribute-like accessor. You access `student.gpa` without parentheses — it looks like an attribute but actually calls the getter method. `@gpa.setter` defines the setter that runs when you write `student.gpa = value`.

`grade_category` is a read-only computed property — it has no setter. Trying to write `student.grade_category = "X"` raises an AttributeError.

Properties let you start with a plain public attribute and later add validation or computation logic without changing how the attribute is accessed externally. The calling code does not need to change from `student.gpa` to `student.get_gpa()`.

In data engineering, properties are used for:

- Validating configuration values when they are set
- Computing derived values on the fly without storing them
- Making certain attributes read-only after object creation

---

### Putting it all together — a realistic data engineering class

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass, field
from typing import Optional


@dataclass
class PipelineStats:
    rows_read: int = 0
    rows_written: int = 0
    rows_failed: int = 0
    errors: list[str] = field(default_factory=list)

    def add_error(self, message: str) -> None:
        self.errors.append(message)
        self.rows_failed += 1

    @property
    def success_rate(self) -> float:
        if self.rows_read == 0:
            return 0.0
        return (self.rows_written / self.rows_read) * 100


class Pipeline(ABC):
    """Abstract base class for all data pipelines."""

    def __init__(self, name: str, batch_size: int = 1000):
        self.name = name
        self.batch_size = batch_size
        self._stats = PipelineStats()
        self._is_running = False

    @abstractmethod
    def extract(self) -> list[dict]:
        """Extract data from the source."""
        pass

    @abstractmethod
    def transform(self, records: list[dict]) -> list[dict]:
        """Apply transformations to the extracted records."""
        pass

    @abstractmethod
    def load(self, records: list[dict]) -> int:
        """Load transformed records to the destination. Returns rows written."""
        pass

    def run(self) -> PipelineStats:
        """Execute the full pipeline: extract, transform, load."""
        print(f"Starting pipeline: {self.name}")
        self._is_running = True

        raw_records = self.extract()
        self._stats.rows_read = len(raw_records)
        print(f"Extracted {self._stats.rows_read} records")

        transformed = self.transform(raw_records)
        print(f"Transformed {len(transformed)} records")

        rows_written = self.load(transformed)
        self._stats.rows_written = rows_written
        print(f"Loaded {rows_written} records")

        self._is_running = False
        print(f"Pipeline complete. Success rate: {self._stats.success_rate:.1f}%")
        return self._stats

    @property
    def stats(self) -> PipelineStats:
        return self._stats

    def __repr__(self) -> str:
        return f"Pipeline(name={self.name!r}, batch_size={self.batch_size})"


class StudentCleaningPipeline(Pipeline):
    """Cleans student data by normalizing names and validating GPAs."""

    def __init__(self):
        super().__init__(name="student_cleaning", batch_size=500)
        self._raw_data = [
            {"student_id": "1", "name": "  arjun mehta  ", "gpa": "8.5"},
            {"student_id": "2", "name": "PRIYA SHARMA",    "gpa": "9.1"},
            {"student_id": "3", "name": "rohan verma",     "gpa": "15.0"},  # invalid
            {"student_id": "4", "name": "sneha iyer",      "gpa": ""},       # missing
        ]

    def extract(self) -> list[dict]:
        return self._raw_data

    def transform(self, records: list[dict]) -> list[dict]:
        cleaned = []
        for record in records:
            try:
                gpa_str = record["gpa"].strip()
                gpa = float(gpa_str) if gpa_str else None
                if gpa is not None and not (0.0 <= gpa <= 10.0):
                    raise ValueError(f"Invalid GPA: {gpa}")
                cleaned.append({
                    "student_id": int(record["student_id"]),
                    "name": " ".join(record["name"].split()).title(),
                    "gpa": gpa,
                })
            except ValueError as e:
                self._stats.add_error(
                    f"Row {record['student_id']}: {e}"
                )
        return cleaned

    def load(self, records: list[dict]) -> int:
        for record in records:
            print(f"  Saving: {record}")
        return len(records)


pipeline = StudentCleaningPipeline()
stats = pipeline.run()

print(f"\nErrors encountered: {len(stats.errors)}")
for error in stats.errors:
    print(f"  - {error}")
```

Run this. Read the output carefully. Trace through the code and make sure you understand why each line produces the output it does.

This is what real data engineering code looks like. A base class defines the contract and the orchestration. Subclasses implement the specific logic. A dataclass holds the statistics. Properties provide controlled access. Abstract methods enforce that every pipeline has extract, transform, and load.

---

### Exercises — Lesson 2

1. Create an abstract base class `DataValidator` with an abstract method `validate(value) -> bool` and a non-abstract method `validate_batch(values: list) -> dict` that returns a dictionary with keys `valid` (list of valid values) and `invalid` (list of invalid values). Implement two concrete subclasses — `RangeValidator` that checks if a number is within a min-max range, and `LengthValidator` that checks if a string's length is within a min-max range. Test both with sample data.
    
2. Create a dataclass `PipelineConfig` with fields: name (str), source_path (str), output_path (str), delimiter (str, default comma), batch_size (int, default 1000), tags (list of str, empty by default). Add a property `is_valid` that returns True only if name, source_path, and output_path are all non-empty strings. Add a method `to_dict` that returns the config as a plain dictionary.
    
3. Create a class `SchemaRegistry` that maintains a class-level dictionary of registered schemas. It should have a class method `register(schema_name: str, columns: dict)` that adds a schema to the registry, a class method `get(schema_name: str)` that retrieves one, and a static method `validate_column_type(type_str: str) -> bool` that checks if a type string is one of a valid set (INTEGER, VARCHAR, DECIMAL, BOOLEAN, DATE, TIMESTAMP). Test by registering two schemas and retrieving them.
    

---

### Answers — Lesson 2

**Exercise 1**

```python
from abc import ABC, abstractmethod

class DataValidator(ABC):
    
    @abstractmethod
    def validate(self, value) -> bool:
        pass

    def validate_batch(self, values: list) -> dict:
        valid = []
        invalid = []
        for value in values:
            if self.validate(value):
                valid.append(value)
            else:
                invalid.append(value)
        return {"valid": valid, "invalid": invalid}


class RangeValidator(DataValidator):
    def __init__(self, min_value: float, max_value: float):
        self.min_value = min_value
        self.max_value = max_value

    def validate(self, value) -> bool:
        try:
            return self.min_value <= float(value) <= self.max_value
        except (TypeError, ValueError):
            return False


class LengthValidator(DataValidator):
    def __init__(self, min_length: int, max_length: int):
        self.min_length = min_length
        self.max_length = max_length

    def validate(self, value) -> bool:
        if not isinstance(value, str):
            return False
        return self.min_length <= len(value) <= self.max_length


gpa_validator = RangeValidator(0.0, 10.0)
print(gpa_validator.validate_batch([8.5, 9.1, 11.0, -1.0, 7.8]))

name_validator = LengthValidator(2, 50)
print(name_validator.validate_batch(["Arjun", "A", "Priya Sharma", "", "X" * 51]))
```

**Exercise 2**

```python
from dataclasses import dataclass, field

@dataclass
class PipelineConfig:
    name: str
    source_path: str
    output_path: str
    delimiter: str = ","
    batch_size: int = 1000
    tags: list[str] = field(default_factory=list)

    @property
    def is_valid(self) -> bool:
        return all([
            bool(self.name.strip()),
            bool(self.source_path.strip()),
            bool(self.output_path.strip()),
        ])

    def to_dict(self) -> dict:
        return {
            "name": self.name,
            "source_path": self.source_path,
            "output_path": self.output_path,
            "delimiter": self.delimiter,
            "batch_size": self.batch_size,
            "tags": self.tags,
        }


config = PipelineConfig(
    name="sales_pipeline",
    source_path="/data/sales.csv",
    output_path="/output/sales.parquet",
    tags=["daily", "sales"]
)
print(config)
print(f"Valid: {config.is_valid}")
print(config.to_dict())

empty_config = PipelineConfig(name="", source_path="/data/x.csv", output_path="/out")
print(f"Empty name valid: {empty_config.is_valid}")
```

**Exercise 3**

```python
class SchemaRegistry:
    _registry: dict[str, dict[str, str]] = {}
    _valid_types = {"INTEGER", "VARCHAR", "DECIMAL", "BOOLEAN", "DATE", "TIMESTAMP"}

    @classmethod
    def register(cls, schema_name: str, columns: dict[str, str]) -> None:
        invalid = {
            col: t for col, t in columns.items()
            if not cls.validate_column_type(t)
        }
        if invalid:
            raise ValueError(f"Invalid column types: {invalid}")
        cls._registry[schema_name] = columns
        print(f"Registered schema: {schema_name}")

    @classmethod
    def get(cls, schema_name: str) -> dict[str, str]:
        if schema_name not in cls._registry:
            raise KeyError(f"Schema '{schema_name}' not found in registry")
        return cls._registry[schema_name]

    @staticmethod
    def validate_column_type(type_str: str) -> bool:
        base_type = type_str.upper().split("(")[0]
        return base_type in SchemaRegistry._valid_types


SchemaRegistry.register("students", {
    "student_id": "INTEGER",
    "name": "VARCHAR(100)",
    "gpa": "DECIMAL(3,2)",
})

SchemaRegistry.register("courses", {
    "course_id": "INTEGER",
    "course_name": "VARCHAR(100)",
    "credits": "INTEGER",
})

print(SchemaRegistry.get("students"))
print(SchemaRegistry.get("courses"))
print(SchemaRegistry.validate_column_type("VARCHAR(50)"))
print(SchemaRegistry.validate_column_type("BLOB"))

try:
    SchemaRegistry.register("bad", {"column": "BLOB"})
except ValueError as e:
    print(e)
```

Lesson 2 complete. Tell me when you are ready for Lesson 3 — File I/O and Data Formats.