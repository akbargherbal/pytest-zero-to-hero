# Chapter 5: Parameterized Testing

## Testing Multiple Scenarios Without Repetition

## Testing Multiple Scenarios Without Repetition

So far, we've written tests that check a single scenario. A function is called with one set of inputs, and we assert one specific outcome. But what happens when a function has different behaviors based on its input?

Consider a simple function that checks if a number is even.

```python
# src/utils.py

def is_even(number):
    """Returns True if a number is even, False otherwise."""
    if not isinstance(number, int):
        raise TypeError("Input must be an integer")
    return number % 2 == 0
```

To test this thoroughly, we need to check several cases:
- A positive even number (e.g., 2)
- A positive odd number (e.g., 3)
- Zero (which is even)
- A negative even number (e.g., -4)
- A negative odd number (e.g., -5)

### The Wrong Way: Repetition

A beginner's first instinct is often to copy and paste the test, changing the values for each scenario. This is a classic example of "illuminating the path by showing the pits." Let's see what this anti-pattern looks like.

```python
# tests/test_utils_repetitive.py

from src.utils import is_even

def test_is_even_with_positive_even():
    assert is_even(2) is True

def test_is_even_with_positive_odd():
    assert is_even(3) is False

def test_is_even_with_zero():
    assert is_even(0) is True

def test_is_even_with_negative_even():
    assert is_even(-4) is True

def test_is_even_with_negative_odd():
    assert is_even(-5) is False
```

Let's run these tests.

```bash
$ pytest -v tests/test_utils_repetitive.py
========================= test session starts ==========================
...
collected 5 items

tests/test_utils_repetitive.py::test_is_even_with_positive_even PASSED [ 20%]
tests/test_utils_repetitive.py::test_is_even_with_positive_odd PASSED  [ 40%]
tests/test_utils_repetitive.py::test_is_even_with_zero PASSED         [ 60%]
tests/test_utils_repetitive.py::test_is_even_with_negative_even PASSED [ 80%]
tests/test_utils_repetitive.py::test_is_even_with_negative_odd PASSED [100%]

========================== 5 passed in ...s ============================
```

The tests pass, but this approach has serious flaws:

1.  **Code Duplication**: The test logic (`assert is_even(...)`) is repeated in every single function. If we need to change the logic, we have to change it in five places.
2.  **Poor Readability**: It's hard to see all the test cases at a glance. You have to read through five separate function definitions to understand the range of inputs being tested.
3.  **Maintenance Burden**: Adding a new test case requires writing a whole new function. This discourages comprehensive testing.

This violates the fundamental programming principle of **DRY (Don't Repeat Yourself)**. There must be a better way. Pytest provides an elegant solution to this exact problem: **parameterization**. The core idea is to write the test logic *once* and supply it with a list of different inputs and expected outputs.

## @pytest.mark.parametrize Basics

## @pytest.mark.parametrize Basics

The `@pytest.mark.parametrize` decorator is the heart of parameterized testing in pytest. It allows you to define multiple sets of arguments and expected outcomes for a single test function.

Let's refactor our repetitive `is_even` tests into one clean, parameterized test.

### The Mental Model Before the Syntax

Before we write the code, let's establish the concept. We want to tell pytest: "Hey, I have this one test function, `test_is_even`. I want you to run it multiple times. For the first run, give it the number `2` and expect `True`. For the second run, give it `3` and expect `False`," and so on.

The `@pytest.mark.parametrize` decorator is how we provide this list of instructions.

### Implementation

Here is the same set of tests, rewritten using parameterization.

```python
# tests/test_utils_parameterized.py

import pytest
from src.utils import is_even

@pytest.mark.parametrize("number, expected", [
    (2, True),
    (3, False),
    (0, True),
    (-4, True),
    (-5, False),
])
def test_is_even(number, expected):
    """Test is_even with multiple integer inputs."""
    assert is_even(number) == expected
```

Let's break this down:

1.  `@pytest.mark.parametrize(...)`: This is the decorator that signals to pytest that this is a parameterized test.
2.  `"number, expected"`: The first argument is a string containing the names of the parameters we are defining, separated by commas. These names **must** match the argument names in our test function signature.
3.  `[...]`: The second argument is a list. Each item in the list represents one full run of the test function.
4.  `(2, True)`, `(3, False)`, etc.: Since we defined two parameter names (`number` and `expected`), each item in our list is a tuple with two values. In the first run, `number` will be `2` and `expected` will be `True`. In the second run, `number` will be `3` and `expected` will be `False`, and so on.
5.  `def test_is_even(number, expected):`: The test function signature now includes the parameter names we defined in the decorator. Pytest will inject the values from our list into these arguments for each run.

Now, let's run this single test function.

```bash
$ pytest -v tests/test_utils_parameterized.py
========================= test session starts ==========================
...
collected 5 items

tests/test_utils_parameterized.py::test_is_even[2-True] PASSED       [ 20%]
tests/test_utils_parameterized.py::test_is_even[3-False] PASSED      [ 40%]
tests/test_utils_parameterized.py::test_is_even[0-True] PASSED       [ 60%]
tests/test_utils_parameterized.py::test_is_even[-4-True] PASSED      [ 80%]
tests/test_utils_parameterized.py::test_is_even[-5-False] PASSED     [100%]

========================== 5 passed in ...s ============================
```

Look at the output! Pytest is smart. It discovered our single test function but understood from the decorator that it represents five distinct test cases. It ran the test five times, each with a different set of parameters.

Notice the test identifiers in the output: `test_is_even[2-True]`, `test_is_even[3-False]`, etc. Pytest automatically generates these IDs from the parameter values to help you identify exactly which case failed.

This single parameterized test is vastly superior to our five repetitive functions. It's concise, easy to read, and simple to maintain. Adding a new test case is as easy as adding a new tuple to the list.

## Multiple Parameters

## Multiple Parameters

The previous example used two parameters, but you can use as many as you need. This is extremely useful for testing functions that take multiple inputs.

Let's define a simple `add` function and test it with various combinations of positive, negative, and zero values.

```python
# src/calculator.py

def add(a, b):
    """Adds two numbers."""
    return a + b
```

Now, let's write a parameterized test for it. We need three parameters for our test: `a`, `b`, and the `expected` result.

```python
# tests/test_calculator.py

import pytest
from src.calculator import add

@pytest.mark.parametrize("a, b, expected", [
    # Positive numbers
    (1, 2, 3),
    (5, 5, 10),
    # Negative numbers
    (-1, -1, -2),
    (-5, 5, 0),
    # Zero
    (0, 0, 0),
    (10, 0, 10),
    # Floating point numbers
    (2.5, 2.5, 5.0)
])
def test_add(a, b, expected):
    """Test the add function with various inputs."""
    assert add(a, b) == expected
```

The structure is identical to our previous example, just with more items in the parameter name string and in each tuple.

1.  `"a, b, expected"`: We define three parameter names.
2.  `(1, 2, 3)`: Each tuple in our list now contains three values, corresponding to `a`, `b`, and `expected` respectively.
3.  `def test_add(a, b, expected):`: The test function accepts all three arguments.

Running this test shows how cleanly pytest handles multiple parameters.

```bash
$ pytest -v tests/test_calculator.py
========================= test session starts ==========================
...
collected 7 items

tests/test_calculator.py::test_add[1-2-3] PASSED                   [ 14%]
tests/test_calculator.py::test_add[5-5-10] PASSED                  [ 28%]
tests/test_calculator.py::test_add[-1--1--2] PASSED                [ 42%]
tests/test_calculator.py::test_add[-5-5-0] PASSED                  [ 57%]
tests/test_calculator.py::test_add[0-0-0] PASSED                   [ 71%]
tests/test_calculator.py::test_add[10-0-10] PASSED                 [ 85%]
tests/test_calculator.py::test_add[2.5-2.5-5.0] PASSED              [100%]

========================== 7 passed in ...s ============================
```

This pattern of `(input1, input2, ..., expected_output)` is one of the most common and powerful uses of parameterization. It allows you to create a comprehensive table of test cases that is both human-readable and machine-executable.

## Combining Parametrization with Fixtures

## Combining Parametrization with Fixtures

Parameterization and fixtures are two of pytest's most powerful features, and they work together beautifully. You can use a fixture to set up a common object or state, and then use parameterization to test various interactions with that object.

Let's imagine we have a `ShoppingCart` class. We want to test that adding various items correctly updates the total price.

```python
# src/shopping_cart.py

class ShoppingCart:
    def __init__(self):
        self.items = {}

    def add_item(self, item_name, price, quantity=1):
        if item_name in self.items:
            self.items[item_name]["quantity"] += quantity
        else:
            self.items[item_name] = {"price": price, "quantity": quantity}

    @property
    def total(self):
        return sum(
            item["price"] * item["quantity"] for item in self.items.values()
        )
```

We can create a fixture that provides a fresh, empty `ShoppingCart` instance for each test run. Then, we can parameterize the test to add different items and check the expected total.

```python
# tests/test_shopping_cart.py

import pytest
from src.shopping_cart import ShoppingCart

@pytest.fixture
def cart():
    """Provides an empty ShoppingCart instance."""
    return ShoppingCart()

@pytest.mark.parametrize("item, price, quantity, expected_total", [
    ("apple", 0.5, 2, 1.0),
    ("banana", 0.75, 1, 0.75),
    ("milk", 3.0, 1, 3.0),
])
def test_add_item_updates_total(cart, item, price, quantity, expected_total):
    """Test that adding an item correctly updates the cart total."""
    cart.add_item(item, price, quantity)
    assert cart.total == expected_total
```

Let's analyze how pytest executes this:

1.  Pytest sees `test_add_item_updates_total` and notices it needs the `cart` fixture and the parameterized arguments.
2.  It also sees the `@pytest.mark.parametrize` decorator with three test cases.
3.  **For the first case (`"apple"`)**:
    a. Pytest executes the `cart()` fixture, creating a new, empty `ShoppingCart` instance.
    b. It calls `test_add_item_updates_total(cart=..., item="apple", price=0.5, quantity=2, expected_total=1.0)`.
    c. The test runs and passes.
4.  **For the second case (`"banana"`)**:
    a. Pytest executes the `cart()` fixture *again*, creating another new, empty `ShoppingCart` instance. This is crucial for test isolation.
    b. It calls `test_add_item_updates_total(cart=..., item="banana", price=0.75, quantity=1, expected_total=0.75)`.
    c. The test runs and passes.
5.  The process repeats for the third case.

The key takeaway is that the fixture's scope (`function` by default) is respected for each parameterized run. Each run is a completely independent test, receiving its own fresh fixture instance. This combination allows you to create complex test scenarios in a very clean and maintainable way.

## Indirect Parametrization

## Indirect Parametrization

Sometimes, the parameters themselves aren't the final data you need for your test. Instead, a parameter might be an identifier or a piece of configuration that you need to use to *generate* the actual test data.

For example, imagine testing a system that deals with different user roles (`'admin'`, `'guest'`, `'editor'`). Your test might not want the string `'admin'`, but rather a fully-formed `User` object with admin privileges.

You could create the object inside the test function, but that mixes setup logic with test logic. A cleaner way is **indirect parametrization**.

### The Problem: Setup Based on a Parameter

Let's define a simple `User` class and a function that greets them.

```python
# src/auth.py

class User:
    def __init__(self, name, role):
        self.name = name
        self.role = role

    def __repr__(self):
        return f"User(name='{self.name}', role='{self.role}')"

def get_greeting(user: User):
    if user.role == "admin":
        return f"Hello, {user.name}! Welcome, administrator."
    return f"Hello, {user.name}."
```

Now, let's write a test. We want to parameterize it with user roles. The "naive" approach would be to create the `User` object inside the test.

```python
# tests/test_auth_naive.py

import pytest
from src.auth import User, get_greeting

@pytest.mark.parametrize("role, expected_greeting", [
    ("admin", "Hello, TestAdmin! Welcome, administrator."),
    ("guest", "Hello, TestGuest."),
])
def test_get_greeting_naive(role, expected_greeting):
    # Setup logic is mixed with test logic here
    if role == "admin":
        user = User("TestAdmin", "admin")
    else:
        user = User("TestGuest", "guest")
    
    assert get_greeting(user) == expected_greeting
```

This works, but it's not ideal. The test function is doing setup work. We can do better by telling pytest to pass our parameter through a fixture first.

### The Solution: `indirect=True`

Indirect parametrization lets you apply a fixture to a specific parameter. When you mark a parameter as `indirect`, pytest will:
1. Take the value for that parameter from your list (e.g., the string `'admin'`).
2. Find a fixture with the **same name** as the parameter.
3. Call that fixture, passing the parameter's value into it via `request.param`.
4. Pass the *result* of the fixture to the test function.

Let's refactor our test using this pattern.

```python
# tests/test_auth_indirect.py

import pytest
from src.auth import User, get_greeting

@pytest.fixture
def user(request):
    """Fixture to create a User object based on the parameter."""
    role = request.param
    if role == "admin":
        return User("TestAdmin", "admin")
    elif role == "guest":
        return User("TestGuest", "guest")
    # You could add more roles here
    return User("DefaultUser", "default")

@pytest.mark.parametrize(
    "user, expected_greeting",
    [
        ("admin", "Hello, TestAdmin! Welcome, administrator."),
        ("guest", "Hello, TestGuest."),
    ],
    indirect=["user"]  # Apply the 'user' fixture to the 'user' parameter
)
def test_get_greeting_indirect(user, expected_greeting):
    """Test get_greeting with user objects created via an indirect fixture."""
    # The 'user' argument is now a User object, not a string!
    assert get_greeting(user) == expected_greeting
```

Let's trace the execution for the first parameter set: `("admin", "...")`.

1.  Pytest sees `indirect=["user"]`. This tells it to treat the `user` parameter specially.
2.  It takes the value for the `user` parameter, which is the string `"admin"`.
3.  It finds the fixture named `user()`.
4.  It calls `user(request)`, where `request.param` is now `"admin"`.
5.  The fixture returns a `User("TestAdmin", "admin")` object.
6.  This `User` object is passed as the `user` argument to `test_get_greeting_indirect()`.
7.  The `expected_greeting` parameter is passed directly as `"Hello, TestAdmin! ..."`.
8.  The assertion `assert get_greeting(user_object) == expected_greeting` is executed.

This pattern is incredibly powerful for separating complex test data setup from the test logic itself, leading to much cleaner and more maintainable tests.

## Generating Test IDs for Clarity

## Generating Test IDs for Clarity

As we saw earlier, pytest automatically generates test IDs for parameterized runs, like `test_add[1-2-3]`. While helpful, these can become unreadable with complex data, such as long strings, nested objects, or non-obvious values.

A failing test report should be a clear map to the problem. Cryptic test IDs are like a map with no labels.

Let's imagine a test for a function that formats a user's data into a display string.

```python
# src/formatter.py

def format_user_display(user_data):
    """Formats user data dict into a display string."""
    if user_data.get("is_admin"):
        return f"{user_data['name']} (Admin)"
    return f"{user_data['name']}"
```

Now, a parameterized test with dictionary inputs.

```python
# tests/test_formatter.py

import pytest
from src.formatter import format_user_display

@pytest.mark.parametrize("user_data, expected", [
    ({"name": "Alice", "is_admin": False}, "Alice"),
    ({"name": "Bob", "is_admin": True}, "Bob (Admin)"),
    ({"name": "Charlie"}, "Charlie"), # Missing is_admin key
])
def test_format_user_display(user_data, expected):
    assert format_user_display(user_data) == expected
```

The output from running this is not very descriptive.

```bash
$ pytest -v tests/test_formatter.py
========================= test session starts ==========================
...
collected 3 items

tests/test_formatter.py::test_format_user_display[user_data0-Alice] PASSED [ 33%]
tests/test_formatter.py::test_format_user_display[user_data1-Bob (Admin)] PASSED [ 66%]
tests/test_formatter.py::test_format_user_display[user_data2-Charlie] PASSED [100%]

========================== 3 passed in ...s ============================
```

`user_data0`, `user_data1`, and `user_data2` tell us nothing about the scenario being tested. If one of these failed, we'd have to go back to the code to figure out which case it was.

### Custom IDs with the `ids` Argument

`@pytest.mark.parametrize` accepts an optional `ids` argument, which is a list of strings to use as test IDs. The list must be the same length as your parameter list.

```python
# tests/test_formatter_with_ids.py

import pytest
from src.formatter import format_user_display

@pytest.mark.parametrize("user_data, expected", [
    ({"name": "Alice", "is_admin": False}, "Alice"),
    ({"name": "Bob", "is_admin": True}, "Bob (Admin)"),
    ({"name": "Charlie"}, "Charlie"),
], ids=[
    "Regular User",
    "Admin User",
    "User with missing admin key"
])
def test_format_user_display_with_ids(user_data, expected):
    assert format_user_display(user_data) == expected
```

Now, the test output is a story, not a puzzle.

```bash
$ pytest -v tests/test_formatter_with_ids.py
========================= test session starts ==========================
...
collected 3 items

tests/test_formatter_with_ids.py::test_format_user_display_with_ids[Regular User] PASSED [ 33%]
tests/test_formatter_with_ids.py::test_format_user_display_with_ids[Admin User] PASSED [ 66%]
tests/test_formatter_with_ids.py::test_format_user_display_with_ids[User with missing admin key] PASSED [100%]

========================== 3 passed in ...s ============================
```

If the "Admin User" test failed, you would immediately know the context of the failure without cross-referencing the test code. This is treating errors as data.

### Generating IDs Programmatically

Manually writing IDs can be tedious if you have many test cases. For more complex scenarios, you can provide a function to the `ids` argument. This function will be called for each parameter set and should return the string ID for that set.

```python
# tests/test_formatter_with_id_func.py

import pytest
from src.formatter import format_user_display

def user_id_generator(user_data):
    """Generates a descriptive test ID from user data."""
    if user_data.get("is_admin"):
        return f"admin-{user_data['name']}"
    return f"user-{user_data['name']}"

@pytest.mark.parametrize("user_data, expected", [
    ({"name": "Alice", "is_admin": False}, "Alice"),
    ({"name": "Bob", "is_admin": True}, "Bob (Admin)"),
], ids=user_id_generator)
def test_format_user_display_with_id_func(user_data, expected):
    assert format_user_display(user_data) == expected
```

When you provide a function, pytest calls it with each value from the *first* parameter (`user_data` in this case).

```bash
$ pytest -v tests/test_formatter_with_id_func.py
========================= test session starts ==========================
...
collected 2 items

tests/test_formatter_with_id_func.py::test_format_user_display_with_id_func[user-Alice] PASSED [ 50%]
tests/test_formatter_with_id_func.py::test_format_user_display_with_id_func[admin-Bob] PASSED [100%]

========================== 2 passed in ...s ============================
```

Using custom IDs is a hallmark of a mature, maintainable test suite. It transforms your test output from a simple pass/fail log into a rich, descriptive report on your code's behavior.
