# Chapter 10: Testing Synchronous Code

## Testing Functions and Methods

## Testing Functions and Methods

At the heart of any Python application are functions and methods. They are the fundamental units of logic, and learning to test them effectively is the cornerstone of building reliable software. In this section, we'll start with the simplest case—a pure function—and see how the principles extend directly to testing methods on an object.

### The Anatomy of a Function Test

A test for a function follows a simple, universal pattern:

1.  **Arrange**: Set up the necessary preconditions and inputs.
2.  **Act**: Call the function or method you want to test.
3.  **Assert**: Verify that the outcome (the return value, a change in state, etc.) is what you expected.

Let's apply this to a concrete example. Imagine we have a utility function in our application that formats user names for display.

Here's the function we want to test, located in a file named `app/utils.py`:

```python
# app/utils.py

def format_user_display_name(first_name: str, last_name: str) -> str:
    """
    Formats a user's name for display, handling potential whitespace
    and capitalizing both names.
    Example: format_user_display_name("  john ", "DOE") -> "John Doe"
    """
    if not first_name or not last_name:
        return ""
    
    # Clean up whitespace and ensure proper capitalization
    formatted_first = first_name.strip().capitalize()
    formatted_last = last_name.strip().capitalize()
    
    return f"{formatted_first} {formatted_last}"
```

Now, let's write our tests in `tests/test_utils.py`.

### Testing the "Happy Path"

First, we test the most common, expected use case. This is often called the "happy path."

```python
# tests/test_utils.py
from app.utils import format_user_display_name

def test_format_user_display_name_happy_path():
    # Arrange: Define the inputs
    first = "ada"
    last = "lovelace"
    
    # Act: Call the function
    result = format_user_display_name(first, last)
    
    # Assert: Check the output
    assert result == "Ada Lovelace"
```

This test is simple, readable, and directly verifies the function's core purpose.

### Testing Edge Cases

Good tests don't just check the happy path; they probe the boundaries and handle unexpected inputs gracefully. What happens if the inputs have extra whitespace or inconsistent capitalization? Our function's docstring claims to handle this, so let's verify it.

```python
# tests/test_utils.py

# ... (previous test)

def test_format_user_display_name_with_whitespace_and_mixed_case():
    """
    Verify that the function correctly handles leading/trailing whitespace
    and different character cases.
    """
    # Arrange
    first = "  GRACE "
    last = "hopper  "
    
    # Act
    result = format_user_display_name(first, last)
    
    # Assert
    assert result == "Grace Hopper"
```

What about invalid or empty inputs? The function's logic returns an empty string. This is a business rule we must test.

```python
# tests/test_utils.py
import pytest
from app.utils import format_user_display_name

# ... (previous tests)

@pytest.mark.parametrize(
    "first, last, expected",
    [
        ("", "Lovelace", ""),
        ("Ada", "", ""),
        (None, "Hopper", ""), # Assuming the function should handle None
        ("Grace", None, ""),
    ]
)
def test_format_user_display_name_empty_or_none_inputs(first, last, expected):
    """
    Verify that empty or None inputs result in an empty string.
    """
    # The function as written will raise an AttributeError for None.
    # This test reveals a bug or an unhandled case!
    # Let's pretend our goal is to fix the function to handle this.
    # After fixing format_user_display_name to check for None, this test will pass.
    
    # For now, let's test only empty strings which the current code handles.
    if first is None or last is None:
        pytest.skip("Skipping None test until function is updated")

    assert format_user_display_name(first, last) == expected
```

Notice how writing the test for `None` immediately revealed a flaw in our function! It would raise an `AttributeError` because you can't call `.strip()` on `None`. This is a perfect example of **Test-Driven Development (TDD)**: writing a test that fails, then writing the code to make it pass.

### From Functions to Methods

Now, let's consider a method. A method is simply a function that is bound to an instance of a class. Testing it is nearly identical, with one extra step in the "Arrange" phase: creating an instance of the object.

Let's create a simple `User` class in `app/models.py`.

```python
# app/models.py

class User:
    def __init__(self, first_name: str, last_name: str, email: str):
        self.first_name = first_name
        self.last_name = last_name
        self.email = email

    def get_display_name(self) -> str:
        """Returns the formatted full name of the user."""
        if not self.first_name or not self.last_name:
            return ""
        
        return f"{self.first_name.capitalize()} {self.last_name.capitalize()}"

    def get_email_domain(self) -> str:
        """Returns the domain part of the user's email."""
        return self.email.split('@')[1]
```

To test the `get_display_name` method, we follow the same Arrange-Act-Assert pattern.

```python
# tests/test_models.py
from app.models import User

def test_user_get_display_name():
    # Arrange: Create an instance of the User class
    user = User(first_name="ada", last_name="lovelace", email="ada@example.com")
    
    # Act: Call the method on the instance
    display_name = user.get_display_name()
    
    # Assert: Check the result
    assert display_name == "Ada Lovelace"

def test_user_get_email_domain():
    # Arrange
    user = User(first_name="grace", last_name="hopper", email="grace.hopper@navy.mil")
    
    # Act
    domain = user.get_email_domain()
    
    # Assert
    assert domain == "navy.mil"
```

As you can see, the core logic is the same. Whether it's a standalone function or a method on an object, you set up the context, call the code, and assert the result.

## Testing Classes and Object-Oriented Code

## Testing Classes and Object-Oriented Code

Testing a class is more than just testing its methods in isolation. It's about verifying the object's behavior, state, and interactions over its entire lifecycle. A class is a blueprint for objects that hold state, and our tests must confirm that this state is managed correctly.

To explore this, let's build a `ShoppingCart` class. This is a classic example because it involves adding items, removing them, and calculating totals—all actions that modify the object's internal state.

Here is our class in `app/models.py`:

```python
# app/models.py

# ... (User class from before)

class ShoppingCart:
    def __init__(self):
        self._items = {}  # Using a dict to store item_id -> quantity

    @property
    def items(self):
        return self._items.copy() # Return a copy to prevent external modification

    def add_item(self, item_id: str, quantity: int = 1):
        if quantity <= 0:
            raise ValueError("Quantity must be positive.")
        self._items[item_id] = self._items.get(item_id, 0) + quantity

    def remove_item(self, item_id: str, quantity: int = 1):
        if quantity <= 0:
            raise ValueError("Quantity must be positive.")
        if item_id not in self._items or self._items[item_id] < quantity:
            raise ValueError("Not enough items in the cart to remove.")
        
        self._items[item_id] -= quantity
        if self._items[item_id] == 0:
            del self._items[item_id]

    def get_total_items(self) -> int:
        return sum(self._items.values())
```

### The Wrong Way: Repetitive Setup

You might be tempted to write tests like this, creating a new cart inside every single test function.

```python
# tests/test_models.py

# ... (User tests)
from app.models import ShoppingCart

def test_cart_add_item_repetitive():
    # Arrange
    cart = ShoppingCart()
    
    # Act
    cart.add_item("apple", 1)
    
    # Assert
    assert cart.items == {"apple": 1}
    assert cart.get_total_items() == 1

def test_cart_add_multiple_items_repetitive():
    # Arrange
    cart = ShoppingCart()
    
    # Act
    cart.add_item("apple", 1)
    cart.add_item("banana", 2)
    
    # Assert
    assert cart.items == {"apple": 1, "banana": 2}
    assert cart.get_total_items() == 3
```

This works, but it violates the **Don't Repeat Yourself (DRY)** principle. The line `cart = ShoppingCart()` will appear in every test. If the `ShoppingCart` constructor ever changes (e.g., to require a user ID), you'd have to update every single test. This is where fixtures, which you learned about in Chapter 4, become essential.

### The Right Way: Using Fixtures for Clean State

A fixture can provide a clean, empty `ShoppingCart` instance to every test that needs one. This isolates tests from each other and centralizes the setup logic.

Let's create a fixture in `tests/test_models.py`.

```python
# tests/test_models.py
import pytest
from app.models import ShoppingCart, User # Assuming all models are in one file

# ... (User tests)

@pytest.fixture
def empty_cart() -> ShoppingCart:
    """Provides an empty shopping cart for tests."""
    return ShoppingCart()

def test_cart_add_item(empty_cart: ShoppingCart):
    # Arrange: The empty_cart fixture handles this
    
    # Act
    empty_cart.add_item("apple", 1)
    
    # Assert
    assert empty_cart.items == {"apple": 1}
    assert empty_cart.get_total_items() == 1

def test_cart_add_multiple_of_same_item(empty_cart: ShoppingCart):
    # Act
    empty_cart.add_item("apple", 1)
    empty_cart.add_item("apple", 2) # Add more of the same item
    
    # Assert
    assert empty_cart.items == {"apple": 3}
    assert empty_cart.get_total_items() == 3
```

Now our tests are cleaner and focused only on the "Act" and "Assert" steps.

### Testing State Transitions

The most important aspect of testing a class is verifying how its state changes in response to method calls. We need to test sequences of actions.

Let's test adding and then removing an item.

```python
# tests/test_models.py

# ... (previous cart tests)

def test_cart_remove_item(empty_cart: ShoppingCart):
    # Arrange: Add items to the cart first
    empty_cart.add_item("apple", 5)
    empty_cart.add_item("banana", 2)
    
    # Act: Remove some of one item
    empty_cart.remove_item("apple", 3)
    
    # Assert: Check the new state
    assert empty_cart.items == {"apple": 2, "banana": 2}
    assert empty_cart.get_total_items() == 4

def test_cart_remove_all_of_one_item(empty_cart: ShoppingCart):
    # Arrange
    empty_cart.add_item("apple", 5)
    empty_cart.add_item("banana", 2)
    
    # Act: Remove all of one item
    empty_cart.remove_item("apple", 5)
    
    # Assert: The item should be completely gone from the cart
    assert "apple" not in empty_cart.items
    assert empty_cart.items == {"banana": 2}
    assert empty_cart.get_total_items() == 2
```

### Testing for Expected Errors

Our `ShoppingCart` class is designed to raise `ValueError` for invalid operations, like removing an item that isn't there. As we saw in Chapter 7, `pytest.raises` is the perfect tool for this.

```python
# tests/test_models.py

# ... (previous cart tests)

def test_cart_remove_too_many_items_raises_error(empty_cart: ShoppingCart):
    empty_cart.add_item("apple", 2)
    
    with pytest.raises(ValueError) as exc_info:
        empty_cart.remove_item("apple", 3) # Try to remove more than available
        
    assert "Not enough items" in str(exc_info.value)

def test_cart_add_negative_quantity_raises_error(empty_cart: ShoppingCart):
    with pytest.raises(ValueError, match="Quantity must be positive"):
        empty_cart.add_item("apple", -1)
```

By testing the happy paths, state transitions, and error conditions, we build a comprehensive and robust test suite for our class, ensuring it behaves exactly as designed.

## Testing Private Methods (And Why You Might Not Want To)

## Testing Private Methods (And Why You Might Not Want To)

As you build complex classes, you'll often create "helper" methods to break down logic into smaller, manageable pieces. In Python, the convention is to prefix these internal-use-only methods with a single underscore (e.g., `_calculate_tax`). This signals to other developers, "This is an implementation detail, don't rely on it."

A common question then arises: "Should I write tests for my private methods?"

The short answer is: **No, you should test the public interface, not the implementation details.**

Let's explore why this philosophy leads to more robust and maintainable tests.

### The Temptation to Test Everything

Imagine we add a tax calculation feature to our `ShoppingCart`. The implementation uses a private helper method.

```python
# app/models.py

class ShoppingCart:
    # ... (previous methods)

    def __init__(self, tax_rate: float = 0.1):
        self._items = {}
        self._tax_rate = tax_rate

    def _calculate_subtotal(self, prices: dict) -> float:
        """Calculates total price before tax."""
        subtotal = 0.0
        for item_id, quantity in self._items.items():
            subtotal += prices.get(item_id, 0.0) * quantity
        return subtotal

    def get_total_price(self, prices: dict) -> float:
        """Calculates the total price including tax."""
        subtotal = self._calculate_subtotal(prices)
        tax = subtotal * self._tax_rate
        return subtotal + tax
```

You might be tempted to write a test directly for `_calculate_subtotal` to ensure it works correctly. You *can* do this, as Python doesn't truly enforce privacy.

```python
# tests/test_models.py

# ...

def test_cart_private_calculate_subtotal_directly():
    """
    This is an example of a BRITTLE test. Avoid this pattern.
    """
    # Arrange
    cart = ShoppingCart()
    cart.add_item("apple", 2)
    cart.add_item("banana", 3)
    prices = {"apple": 1.0, "banana": 0.5}
    
    # Act: Call the private method directly
    subtotal = cart._calculate_subtotal(prices)
    
    # Assert
    assert subtotal == 3.50 # (2 * 1.0) + (3 * 0.5)
```

This test passes. So what's the problem?

### The Pitfall: Brittle Tests Coupled to Implementation

The problem arises when you refactor your code. Good developers constantly refactor to improve clarity and performance. Let's say you realize the `_calculate_subtotal` logic is simple enough to be inlined directly into `get_total_price`.

```python
# app/models.py (Refactored)

class ShoppingCart:
    # ...

    def get_total_price(self, prices: dict) -> float:
        """Calculates the total price including tax (refactored)."""
        subtotal = 0.0
        for item_id, quantity in self._items.items():
            subtotal += prices.get(item_id, 0.0) * quantity
        
        tax = subtotal * self._tax_rate
        return subtotal + tax
    
    # The _calculate_subtotal method has been removed!
```

The public behavior of `get_total_price` is **exactly the same**. The class works perfectly. But what happens when you run your tests?
```bash
$ pytest tests/test_models.py
...
_________________ test_cart_private_calculate_subtotal_directly __________________

    def test_cart_private_calculate_subtotal_directly():
        # ...
        # Act: Call the private method directly
>       subtotal = cart._calculate_subtotal(prices)
E       AttributeError: 'ShoppingCart' object has no attribute '_calculate_subtotal'

tests/test_models.py:123: AttributeError
```
Your test suite fails! Your test for `_calculate_subtotal` is now broken, even though the class's public functionality is unchanged. This is a **false negative**. The test failed not because of a bug in the application code, but because it was too tightly coupled to a specific implementation detail that was free to change.

### The Solution: Test the Behavior

The correct approach is to test the public method (`get_total_price`) in a way that implicitly covers the logic that *was* in the private method.

```python
# tests/test_models.py

@pytest.fixture
def cart_with_items() -> ShoppingCart:
    """Provides a cart with some items already in it."""
    cart = ShoppingCart(tax_rate=0.1) # Use a known tax rate
    cart.add_item("apple", 2) # 2 * $1.00 = $2.00
    cart.add_item("banana", 3) # 3 * $0.50 = $1.50
    return cart

def test_get_total_price(cart_with_items: ShoppingCart):
    # Arrange
    prices = {"apple": 1.0, "banana": 0.5}
    # Subtotal should be $3.50
    # Tax should be $0.35 (10% of 3.50)
    # Total should be $3.85
    
    # Act
    total = cart_with_items.get_total_price(prices)
    
    # Assert
    assert total == pytest.approx(3.85)
```

This test verifies the correct final price. It doesn't care *how* the subtotal was calculated, only that the final result is correct. Now, you are free to refactor the internal implementation of `ShoppingCart` as much as you want. As long as `get_total_price` returns the correct value, this test will pass.

Your tests now protect the **what** (the object's behavior) and not the **how** (the object's internal implementation), leading to a more resilient and valuable test suite.

## Testing Code with Side Effects

## Testing Code with Side Effects

A "pure function" is a developer's dream: for a given input, it always returns the same output and has no observable effects on the outside world. The `format_user_display_name` function from earlier is a good example.

However, most real-world code is not pure. It has **side effects**: actions that change state outside the function's scope. Common side effects include:

*   Writing to a file or database.
*   Making an HTTP request to an external API.
*   Printing to the console.
*   Modifying a global variable.

Side effects make testing harder because they introduce dependencies on external systems (like the filesystem or the network) and can make tests slow, unreliable, and difficult to set up. The key to testing code with side effects is to **isolate your code from the side effect itself**.

### The Problem: A Function That Writes to a File

Let's consider a function that logs an event message to a file.

```python
# app/logging.py
from datetime import datetime

def log_event(message: str, log_file: str):
    """
    Writes a timestamped event message to a log file.
    """
    timestamp = datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S")
    with open(log_file, "a") as f:
        f.write(f"[{timestamp}] {message}\n")
```

How would we test this? We could run the function and then read the file to see if the message was written.

```python
# tests/test_logging_bad.py
import os
from app.logging import log_event

def test_log_event_writes_to_file_bad_approach(tmp_path):
    """
    This test works, but has several problems.
    """
    # Arrange
    log_file = tmp_path / "events.log"
    
    # Act
    log_event("User logged in", str(log_file))
    
    # Assert
    with open(log_file, "r") as f:
        content = f.read()
    
    assert "User logged in" in content
    
    # Teardown is handled by tmp_path, but in other cases
    # you might need manual cleanup.
    # os.remove(log_file)
```

While pytest's built-in `tmp_path` fixture helps, this approach has drawbacks:
1.  **It's slow**: Filesystem I/O is orders of magnitude slower than in-memory operations.
2.  **It's complex**: The test has to perform file operations just to verify the result.
3.  **It's not a true unit test**: It depends on the filesystem being available and working correctly.

### The Solution: Mocking the Side Effect

The solution is to use a **mock**. As we'll cover in depth in Chapters 8 and 9, a mock is a test double that replaces a real object or function. We can replace the built-in `open` function with a mock object that we control and inspect.

Pytest integrates seamlessly with Python's standard `unittest.mock` library. The `mocker` fixture, provided by the `pytest-mock` plugin (a common dependency), is the standard way to do this.

```bash
# You'll likely need to install this
pip install pytest-mock
```

```python
# tests/test_logging.py
from app.logging import log_event
from unittest.mock import patch, mock_open

def test_log_event_with_mocker(mocker):
    """
    Tests the log_event function by mocking the 'open' call.
    """
    # Arrange: We want to mock 'open' inside the 'app.logging' module
    # where it is being called.
    mock_file = mocker.mock_open()
    mocker.patch("app.logging.open", mock_file)
    
    # Act
    log_event("User logged out", "any/file/path.log")
    
    # Assert: Check that 'open' was called correctly
    mock_file.assert_called_once_with("any/file/path.log", "a")
    
    # Assert: Check that 'write' was called on the file handle
    # We can't easily check the timestamp, so we check for the message.
    # The call is a bit complex: handle.write(string)
    written_content = mock_file().write.call_args[0][0]
    assert "User logged out" in written_content
    assert written_content.endswith("\n")
```

Let's break down what happened:
1.  `mocker.patch("app.logging.open", ...)`: We told pytest to replace the `open` function *within the `app.logging` namespace* with our mock. This is a critical detail—you must patch the object where it is *looked up*, not where it is defined.
2.  `mocker.mock_open()`: This is a convenient helper that creates a mock that behaves like a file handle.
3.  `mock_file.assert_called_once_with(...)`: We assert that the `open` function was called exactly once with the expected filename and mode (`"a"` for append).
4.  `mock_file().write.call_args`: We inspect the arguments that were passed to the `write` method on the mocked file handle to ensure our message was part of it.

This test is fast, runs entirely in memory, and precisely verifies that our function tried to perform the correct side effect without actually doing it. This technique is fundamental for testing code that interacts with databases, APIs, or any other external system.

## Integration Testing Within Your Codebase

## Integration Testing Within Your Codebase

So far, we've focused on **unit tests**, which test a single component (a function or a class) in isolation. This is the foundation of a solid testing strategy. However, a real application is a collection of many units working together. A bug might not be in any single unit, but in the interaction *between* them.

This is where **integration tests** come in. An integration test verifies that two or more components of your application can communicate and work together correctly.

### From Unit to Integration

Let's evolve our `ShoppingCart` example. In a real e-commerce system, product prices aren't hardcoded in the test; they come from a database or another service. Let's create a `ProductDB` class to represent this.

```python
# app/db.py

class ProductDB:
    def __init__(self):
        # In a real app, this would connect to a database.
        # Here, it's just an in-memory dictionary.
        self._prices = {
            "apple": 1.0,
            "banana": 0.5,
            "orange": 0.75,
        }

    def get_price(self, item_id: str) -> float:
        """Returns the price of an item."""
        return self._prices.get(item_id, 0.0)
```

Now, we'll refactor `ShoppingCart` to use this `ProductDB`. This is a common pattern called **Dependency Injection**, where a component's dependencies are provided from the outside rather than created internally.

```python
# app/models.py (Refactored ShoppingCart)
from app.db import ProductDB

class ShoppingCart:
    def __init__(self, db: ProductDB, tax_rate: float = 0.1):
        self._items = {}
        self._db = db
        self._tax_rate = tax_rate

    # ... add_item, remove_item, etc. remain the same ...

    def get_total_price(self) -> float:
        """Calculates the total price using the product database."""
        subtotal = 0.0
        for item_id, quantity in self._items.items():
            price = self._db.get_price(item_id)
            subtotal += price * quantity
        
        tax = subtotal * self._tax_rate
        return subtotal + tax
```

Our `ShoppingCart` now depends on `ProductDB`. We have two units that need to collaborate.

### Path 1: The Unit Test (with Mocks)

We can still write a unit test for `ShoppingCart` in isolation. To do this, we provide a *mock* `ProductDB` instead of a real one. This allows us to test the cart's calculation logic without any dependency on the actual database.

```python
# tests/test_models_integration.py
from unittest.mock import Mock
from app.models import ShoppingCart
from app.db import ProductDB

def test_cart_total_price_unit_test(mocker):
    """
    Unit test for ShoppingCart, mocking the ProductDB dependency.
    """
    # Arrange
    # Create a mock ProductDB that returns specific prices for our test
    mock_db = mocker.Mock(spec=ProductDB)
    mock_db.get_price.side_effect = lambda item_id: {"apple": 1.0, "banana": 0.5}.get(item_id, 0.0)
    
    cart = ShoppingCart(db=mock_db, tax_rate=0.1)
    cart.add_item("apple", 2)
    cart.add_item("banana", 3)
    
    # Act
    total_price = cart.get_total_price()
    
    # Assert
    # Subtotal = (2 * 1.0) + (3 * 0.5) = 3.5
    # Total = 3.5 * 1.1 = 3.85
    assert total_price == pytest.approx(3.85)
    
    # We can also assert that the dependency was called correctly
    assert mock_db.get_price.call_count == 2
    mock_db.get_price.assert_any_call("apple")
    mock_db.get_price.assert_any_call("banana")
```

**Advantages of this unit test:**
*   **Fast**: No real database connection.
*   **Isolated**: A failure here points directly to a bug in `ShoppingCart`, not `ProductDB`.
*   **Controllable**: We can easily simulate any scenario, like an item not being in the database, by configuring our mock.

### Path 2: The Integration Test (with Real Objects)

The unit test gives us confidence that `ShoppingCart`'s logic is correct. But does it work with the *real* `ProductDB`? An integration test will answer that.

For this test, we instantiate *both* real objects and have them interact.

```python
# tests/test_models_integration.py
import pytest
from app.models import ShoppingCart
from app.db import ProductDB

def test_cart_and_db_integration():
    """
    Integration test verifying ShoppingCart and ProductDB work together.
    """
    # Arrange: Create real instances of both classes
    db = ProductDB()
    cart = ShoppingCart(db=db, tax_rate=0.1)
    
    cart.add_item("apple", 2)
    cart.add_item("orange", 4)
    
    # Act
    total_price = cart.get_total_price()
    
    # Assert
    # Subtotal = (2 * 1.0) + (4 * 0.75) = 2.0 + 3.0 = 5.0
    # Total = 5.0 * 1.1 = 5.5
    assert total_price == pytest.approx(5.5)
```

**Advantages of this integration test:**
*   **High Confidence**: It proves the two components are correctly wired together and can communicate. It tests the contract between them.
*   **Realistic**: It more closely simulates how the application will run in production.

### The Testing Pyramid

In a professional project, you need both types of tests. The general guideline is the "Testing Pyramid":

1.  **Lots of fast Unit Tests** at the base, providing detailed coverage of individual components.
2.  **Fewer, slightly slower Integration Tests** in the middle, ensuring components work together.
3.  **Very few End-to-End Tests** at the top, which test the entire application through its user interface.

By testing your synchronous code at both the unit and integration levels, you create a safety net that catches bugs in individual components and in the critical connections between them.
