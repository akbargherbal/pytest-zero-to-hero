# Chapter 10: Testing Synchronous Code

## Testing Functions and Methods

## The Foundation: Testing Pure Functions

At the heart of any application lies logic, encapsulated in functions and methods. The simplest and most reliable code to test is a **pure function**: a function that, for the same input, will always return the same output and has no observable side effects.

We will build our understanding around a common e-commerce scenario: processing a customer's order. This will be our **anchor example** for the entire chapter, evolving from a simple function into a complex class with dependencies.

### Phase 1: Establish the Reference Implementation

Let's start with a simple function to calculate the total price of an order.

First, we need a representation of a product. We'll use a simple data structure.

```python
# src/product.py
from dataclasses import dataclass

@dataclass
class Product:
    name: str
    price: float
    
    def __post_init__(self):
        if self.price < 0:
            raise ValueError("Price cannot be negative.")
```

Now, let's create our core business logic function.

```python
# src/order_processing.py
from .product import Product

def calculate_order_total(product: Product, quantity: int) -> float:
    """Calculates the total price for a given product and quantity."""
    if quantity < 1:
        raise ValueError("Quantity must be at least 1.")
    
    return product.price * quantity
```

This function is a perfect candidate for our first test. It takes inputs, performs a calculation, and returns a value. It doesn't write to a file, access a database, or call an API. It's pure.

Let's write a straightforward pytest test for the "happy path"—a valid product and quantity.

```python
# tests/test_order_processing.py
from src.order_processing import calculate_order_total
from src.product import Product

def test_calculate_order_total_success():
    """
    Tests that the total is calculated correctly for a valid product and quantity.
    """
    # Arrange
    product = Product(name="Laptop", price=1000.0)
    quantity = 3
    
    # Act
    total = calculate_order_total(product, quantity)
    
    # Assert
    assert total == 3000.0
```

Running this test is simple and gives us immediate confidence.

```bash
$ pytest
============================= test session starts ==============================
...
collected 1 item

tests/test_order_processing.py .                                         [100%]

============================== 1 passed in ...s ===============================
```

This is the essence of unit testing: verifying a small, isolated piece of logic.

### Iteration 1: From Function to Method

As our application grows, standalone functions often become methods of a class to manage related state and behavior. Let's refactor our logic into an `OrderProcessor` class. This is a common evolutionary step in software design.

Here is the new class structure. For now, it's just a container for our calculation method.

```python
# src/order_processor.py (New File)
from .product import Product

class OrderProcessor:
    def calculate_order_total(self, product: Product, quantity: int) -> float:
        """Calculates the total price for a given product and quantity."""
        if quantity < 1:
            raise ValueError("Quantity must be at least 1.")
        
        return product.price * quantity
```

How does this change our test? The core logic is the same, but the way we call it is different. We now need an *instance* of the `OrderProcessor` class.

Here is the updated test file.

```python
# tests/test_order_processor.py (New Test File)
from src.order_processor import OrderProcessor
from src.product import Product

def test_calculate_order_total_success():
    """
    Tests that the total is calculated correctly for a valid product and quantity.
    """
    # Arrange
    processor = OrderProcessor()  # We need an instance now
    product = Product(name="Laptop", price=1000.0)
    quantity = 3
    
    # Act
    total = processor.calculate_order_total(product, quantity)
    
    # Assert
    assert total == 3000.0
```

The test still passes, and the change seems trivial. However, this shift from a stateless function to a stateful class is the most significant leap in testing complexity. It opens the door to new challenges related to state management, dependencies, and side effects, which we will tackle in the upcoming sections.

## Testing Classes and Object-Oriented Code

## The Challenge of State

Classes are more than just collections of methods; they encapsulate **state** (data) and **behavior** (methods). While our `OrderProcessor` is currently stateless, any realistic implementation would need to track information, such as a history of processed orders.

Testing stateful objects introduces a critical new requirement: **test isolation**. One test must not be affected by the state changes caused by another.

### Iteration 2: Introducing State and a New Bug

Let's add a feature to our `OrderProcessor`: it will now keep a record of every order ID it processes.

```python
# src/order_processor.py (Updated)
import uuid
from .product import Product

class OrderProcessor:
    def __init__(self):
        self.processed_orders = []

    def _generate_order_id(self) -> str:
        """Generates a unique order ID."""
        return str(uuid.uuid4())

    def process_order(self, product: Product, quantity: int):
        """Processes an order and records it."""
        if quantity < 1:
            raise ValueError("Quantity must be at least 1.")
        
        order_id = self._generate_order_id()
        # In a real system, this would do more (e.g., charge card, update inventory)
        self.processed_orders.append(order_id)
        print(f"Processed order {order_id} for {quantity} of {product.name}.")
        return order_id
```

Our old test for `calculate_order_total` is now obsolete. We need new tests for `process_order` that verify two things:
1.  It returns an order ID (behavior).
2.  It adds the order ID to `processed_orders` (state change).

Let's write two tests. A common but flawed approach for beginners is to use a single, shared instance of the class for all tests.

```python
# tests/test_order_processor.py (Problematic Version)
from src.order_processor import OrderProcessor
from src.product import Product

# Create a single instance to be shared by all tests in this module
processor = OrderProcessor()
product = Product(name="Keyboard", price=75.0)

def test_process_order_adds_to_history():
    """Verify that a processed order is added to the processor's history."""
    initial_history_count = len(processor.processed_orders)
    
    processor.process_order(product, 2)
    
    assert len(processor.processed_orders) == initial_history_count + 1

def test_process_single_item_order():
    """Verify that processing an order for a single item works."""
    # This test has a hidden assumption: that processed_orders is empty.
    assert len(processor.processed_orders) == 0
    
    processor.process_order(product, 1)
    
    assert len(processor.processed_orders) == 1
```

Now, let's run pytest. One of our tests is going to fail.

```bash
$ pytest -v
============================= test session starts ==============================
...
collected 2 items

tests/test_order_processor.py::test_process_order_adds_to_history PASSED [ 50%]
tests/test_order_processor.py::test_process_single_item_order FAILED   [100%]

=================================== FAILURES ===================================
_______________________ test_process_single_item_order _______________________

    def test_process_single_item_order():
        """Verify that processing an order for a single item works."""
        # This test has a hidden assumption: that processed_orders is empty.
>       assert len(processor.processed_orders) == 0
E       assert 1 == 0
E        +  where 1 = len(['...'])

tests/test_order_processor.py:20: AssertionError
=========================== short test summary info ============================
FAILED tests/test_order_processor.py::test_process_single_item_order - assert 1 == 0
========================= 1 failed, 1 passed in ...s =========================
```

### Diagnostic Analysis: Reading the Failure

**The complete output**: The summary shows one test passed and one failed. The failure is `test_process_single_item_order`.

**Let's parse this section by section**:

1.  **The summary line**: `FAILED tests/test_order_processor.py::test_process_single_item_order - assert 1 == 0`
    *   **What this tells us**: The test failed because of an `AssertionError`. The specific assertion that failed was comparing `1` and `0`.

2.  **The traceback**:
    ```python
    def test_process_single_item_order():
        """Verify that processing an order for a single item works."""
        # This test has a hidden assumption: that processed_orders is empty.
>       assert len(processor.processed_orders) == 0
    ```
    *   **What this tells us**: The failure occurred on line 20 of our test file.
    *   **Key line**: `assert len(processor.processed_orders) == 0` is the exact point of failure.

3.  **The assertion introspection**:
    ```
    E       assert 1 == 0
    E        +  where 1 = len(['...'])
    ```
    *   **What this tells us**: Pytest's powerful introspection shows us the *values* at the time of the assertion. The left side of the `==`, which was `len(processor.processed_orders)`, evaluated to `1`. The right side was `0`. The assertion `1 == 0` is correctly identified as false.

**Root cause identified**: The `test_process_single_item_order` test incorrectly assumed that the `processor.processed_orders` list would be empty at the start of the test. It was not; it contained one item.

**Why the current approach can't solve this**: By creating a single `processor` instance at the module level, we created shared state. The first test to run (`test_process_order_adds_to_history`) modified this shared state by adding an order ID to the list. The second test inherited this modified state, causing its initial assumption to be violated. The order of test execution determined the outcome, a classic sign of a fragile test suite.

**What we need**: A mechanism to ensure that every single test function gets a fresh, clean instance of `OrderProcessor`, guaranteeing test isolation.

### The Solution: Fixtures for Isolation

This is precisely the problem that pytest fixtures are designed to solve. A fixture is a function that provides a resource (like a class instance, a database connection, or a dataset) to your tests. Pytest ensures that fixtures are set up and torn down cleanly for each test that uses them.

Let's refactor our tests to use a fixture. By convention, fixtures are often placed in a `conftest.py` file to be shared across multiple test files, but for a single file, defining it locally is fine.

```python
# tests/test_order_processor.py (Corrected Version)
import pytest
from src.order_processor import OrderProcessor
from src.product import Product

@pytest.fixture
def processor():
    """Returns a fresh OrderProcessor instance for each test."""
    return OrderProcessor()

@pytest.fixture
def product():
    """Returns a sample product for tests."""
    return Product(name="Keyboard", price=75.0)

def test_process_order_adds_to_history(processor, product):
    """Verify that a processed order is added to the processor's history."""
    initial_history_count = len(processor.processed_orders)
    assert initial_history_count == 0 # We can now safely assume this
    
    processor.process_order(product, 2)
    
    assert len(processor.processed_orders) == initial_history_count + 1

def test_process_single_item_order(processor, product):
    """Verify that processing an order for a single item works."""
    # This test now gets its own 'processor' instance.
    assert len(processor.processed_orders) == 0
    
    processor.process_order(product, 1)
    
    assert len(processor.processed_orders) == 1
```

Let's run the tests again.

```bash
$ pytest -v
============================= test session starts ==============================
...
collected 2 items

tests/test_order_processor.py::test_process_order_adds_to_history PASSED [ 50%]
tests/test_order_processor.py::test_process_single_item_order PASSED   [100%]

============================== 2 passed in ...s ===============================
```

**Success!** By declaring `processor` and `product` as arguments in our test functions, we told pytest to execute the corresponding fixture functions and pass their return values to our tests. Because the default **scope** of a fixture is `function`, pytest creates a brand new `OrderProcessor` and `Product` for each test, guaranteeing isolation. The `test_process_single_item_order` now receives an instance whose `processed_orders` list is guaranteed to be empty.

This solves the state management problem, but our `OrderProcessor` has a hidden implementation detail (`_generate_order_id`) that we might be tempted to test directly.

## Testing Private Methods (And Why You Might Not Want To)

## The Temptation of Implementation Details

Our `OrderProcessor` has a "private" method, `_generate_order_id`, indicated by the leading underscore. This is a Python convention signifying that the method is an internal implementation detail and not part of the class's public API.

A common question arises: "Should I write a test specifically for `_generate_order_id`?"

The short answer is usually **no**.

### The Philosophy: Test Behavior, Not Implementation

Your tests should act as the first user of your code. A user of the `OrderProcessor` class doesn't care *how* the order ID is generated; they only care that when they call the public method `process_order`, they get a unique ID back and the order is processed correctly.

-   **Testing Public Behavior**: When you test `process_order`, you are implicitly testing `_generate_order_id`. If the ID generation were to break (e.g., return `None`), your test for `process_order` would fail. This is good! The test tells you that a user-facing behavior is broken.
-   **Testing Private Implementation**: If you write a separate test for `_generate_order_id`, you are coupling your test suite to the internal structure of your class.

Let's see what happens when we refactor the implementation without changing the behavior.

### Iteration 3: A Refactor that Breaks a Bad Test

Imagine we decide to change the order ID format to be prefixed with `ORD-`. The public behavior is unchanged—it still returns a unique string.

```python
# src/order_processor.py (Refactored)
import uuid
from .product import Product

class OrderProcessor:
    def __init__(self):
        self.processed_orders = []

    def _generate_order_id(self) -> str:
        """Generates a unique order ID with a prefix."""
        # We changed the implementation detail.
        return f"ORD-{uuid.uuid4()}"

    def process_order(self, product: Product, quantity: int):
        """Processes an order and records it."""
        if quantity < 1:
            raise ValueError("Quantity must be at least 1.")
        
        order_id = self._generate_order_id()
        self.processed_orders.append(order_id)
        print(f"Processed order {order_id} for {quantity} of {product.name}.")
        return order_id
```

Our existing tests for `process_order` still pass perfectly because they only check that an ID was added to the history. They don't care about the ID's format. This is the hallmark of a robust test suite.

Now, let's demonstrate the **wrong way**. Suppose we had written a test that was tightly coupled to the implementation of `_generate_order_id`.

```python
# tests/test_order_processor_bad.py (A new file demonstrating a bad practice)
import uuid
from src.order_processor import OrderProcessor

def test_generate_order_id_is_valid_uuid():
    """
    This is a BRITTLE test. It tests a private implementation detail.
    """
    processor = OrderProcessor()
    # We access the private method directly
    order_id = processor._generate_order_id()
    
    # This assertion is tied to the OLD implementation.
    # It assumes the ID is a pure UUID string.
    try:
        uuid.UUID(order_id)
    except ValueError:
        pytest.fail(f"'{order_id}' is not a valid UUID.")
```

With our refactored `OrderProcessor`, this new "bad" test will now fail.

```bash
$ pytest tests/test_order_processor_bad.py
============================= test session starts ==============================
...
collected 1 item

tests/test_order_processor_bad.py F                                      [100%]

=================================== FAILURES ===================================
______________________ test_generate_order_id_is_valid_uuid ______________________

    def test_generate_order_id_is_valid_uuid():
        """
        This is a BRITTLE test. It tests a private implementation detail.
        """
        processor = OrderProcessor()
        # We access the private method directly
        order_id = processor._generate_order_id()
    
        # This assertion is tied to the OLD implementation.
        # It assumes the ID is a pure UUID string.
        try:
            uuid.UUID(order_id)
        except ValueError:
>           pytest.fail(f"'{order_id}' is not a valid UUID.")
E           Failed: 'ORD-...' is not a valid UUID.

tests/test_order_processor_bad.py:17: Failed
=========================== short test summary info ============================
FAILED tests/test_order_processor_bad.py::test_generate_order_id_is_valid_uuid
============================== 1 failed in ...s ===============================
```

### Diagnostic Analysis: The Cost of Brittle Tests

The test failed because the code worked as intended! We changed an internal detail (`_generate_order_id`) without breaking the public contract (`process_order`). The test failed not because of a bug in the application code, but because the test itself was too specific and fragile.

**Root cause identified**: The test was coupled to the implementation, not the behavior.

**Why this is a problem**:
1.  **Increased Maintenance**: Every time you refactor internal code, you have to fix a series of failing tests, even if the public behavior is unchanged. This slows down development and discourages refactoring.
2.  **Reduced Clarity**: The test failure doesn't indicate a real bug. It's noise that hides real problems.

### When to Reconsider

There are exceptions. If a private method contains extremely complex, critical logic (e.g., a sophisticated pricing algorithm), you might argue for testing it directly. However, a better approach is often to ask: "If this logic is so important, why is it a private method?"

Often, the need to test a private method is a "code smell" suggesting that the class is doing too much. The complex logic might be better off extracted into its own, separate class or module. That new component would have a public API, which you could then test thoroughly and with confidence.

**Guideline**: Start by testing only public methods. If you feel a strong urge to test a private one, first consider refactoring your code to make that logic part of a public API on another, more focused object.

## Testing Code with Side Effects

## The World Outside Your Function

So far, our `OrderProcessor` has lived in a vacuum. A real-world order processor must interact with other systems: it needs to check inventory, charge a credit card, and maybe send an email. These interactions are called **side effects**.

Side effects make testing dramatically harder. If your test debits a real inventory database or calls a real payment API, it becomes:
*   **Slow**: Network calls and database queries are orders of magnitude slower than in-memory operations.
*   **Expensive**: You might be charged for API calls.
*   **Unreliable**: The test could fail due to network issues or a third-party service being down.
*   **Destructive**: The test changes the state of an external system, violating test isolation.

The solution is to isolate our code from its dependencies during testing using **Test Doubles**.

### Iteration 4: Introducing Dependencies and Side Effects

Let's make our `OrderProcessor` more realistic. It will now depend on an `InventorySystem` and a `PaymentGateway`. We'll use the principle of **Dependency Injection**: instead of creating its dependencies, the `OrderProcessor` will receive them in its constructor. This is crucial for testability.

```python
# src/dependencies.py (New File)
from .product import Product

class InventorySystem:
    def __init__(self):
        # In a real app, this would connect to a database.
        self._stock = {
            "Laptop": 10,
            "Keyboard": 25,
        }

    def check_stock(self, product: Product, quantity: int) -> bool:
        print(f"DATABASE: Checking stock for {product.name}")
        return self._stock.get(product.name, 0) >= quantity

    def reduce_stock(self, product: Product, quantity: int):
        print(f"DATABASE: Reducing stock for {product.name} by {quantity}")
        self._stock[product.name] -= quantity

class PaymentGateway:
    def charge(self, amount: float, card_details: str) -> bool:
        print(f"API: Charging card {card_details} for ${amount}")
        # In a real app, this would call a third-party API like Stripe.
        if not card_details:
            return False
        return True
```

```python
# src/order_processor.py (Updated with Dependencies)
import uuid
from .product import Product
from .dependencies import InventorySystem, PaymentGateway

class OrderProcessor:
    def __init__(self, inventory: InventorySystem, payment: PaymentGateway):
        self.inventory = inventory
        self.payment = payment
        self.processed_orders = []

    def _generate_order_id(self) -> str:
        return f"ORD-{uuid.uuid4()}"

    def process_order(self, product: Product, quantity: int, card_details: str):
        if not self.inventory.check_stock(product, quantity):
            raise ValueError("Not enough stock.")
        
        total_price = product.price * quantity
        if not self.payment.charge(total_price, card_details):
            raise ValueError("Payment failed.")
            
        self.inventory.reduce_stock(product, quantity)
        
        order_id = self._generate_order_id()
        self.processed_orders.append(order_id)
        return order_id
```

Now, let's write a test for the happy path. A naive approach would be to use the real dependencies.

```python
# tests/test_order_processor.py (New test with real dependencies)
# ... (previous fixtures and tests) ...

from src.dependencies import InventorySystem, PaymentGateway

def test_process_order_with_real_dependencies_fails_on_second_run():
    # Arrange
    inventory = InventorySystem() # Real dependency
    payment = PaymentGateway()    # Real dependency
    processor = OrderProcessor(inventory, payment)
    product = Product("Laptop", 1000.0)

    # Act
    processor.process_order(product, quantity=8, card_details="1234-5678")

    # Assert
    # We can't easily check the database, but we can check our own state
    assert len(processor.processed_orders) == 1
    # Let's try to check the inventory state
    assert not inventory.check_stock(product, 5) # 10 - 8 = 2 left, so 5 is not available
```

This test might pass the first time. But what happens if we run it twice? Or what if we run another test that also uses the "Laptop" inventory? Let's run this specific test twice in a row using `pytest -k ... -v --count=2`.

```bash
$ pytest -k test_process_order_with_real_dependencies -v --count=2
============================= test session starts ==============================
...
collected 1 item / 1 deselected / 1 selected

tests/test_order_processor.py::test_process_order_with_real_dependencies_fails_on_second_run [1/2] PASSED [ 50%]
tests/test_order_processor.py::test_process_order_with_real_dependencies_fails_on_second_run [2/2] FAILED [100%]

=================================== FAILURES ===================================
_ test_process_order_with_real_dependencies_fails_on_second_run _

    def test_process_order_with_real_dependencies_fails_on_second_run():
        # ...
        processor = OrderProcessor(inventory, payment)
        product = Product("Laptop", 1000.0)
    
        # Act
>       processor.process_order(product, quantity=8, card_details="1234-5678")

tests/test_order_processor.py:50: 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
src/order_processor.py:19: in process_order
    raise ValueError("Not enough stock.")
E   ValueError: Not enough stock.
=========================== short test summary info ============================
FAILED tests/test_order_processor.py::test_process_order_with_real_dependencies_fails_on_second_run - ValueError: Not enough stock.
========================= 1 failed, 1 passed in ...s =========================
```

### Diagnostic Analysis: Reading the Failure

**The complete output**: The first run passed, but the second run failed with a `ValueError`.

**Let's parse this section by section**:

1.  **The summary line**: `FAILED ... - ValueError: Not enough stock.`
    *   **What this tells us**: The test failed because our application code raised a `ValueError` with a specific message. This wasn't an `AssertionError`; the test crashed before it could even get to the assertion.

2.  **The traceback**:
    ```python
    src/order_processor.py:19: in process_order
        raise ValueError("Not enough stock.")
    ```
    *   **What this tells us**: The crash happened inside our application code, in the `process_order` method, when it tried to raise the "Not enough stock" error. This means the `inventory.check_stock(product, quantity)` call must have returned `False`.

**Root cause identified**: The first test run reduced the stock of "Laptop" from 10 to 2. The second test run tried to order 8 more, but since only 2 were available, `check_stock` returned `False`, and our code correctly raised an exception. The test failed because of a side effect from a previous run.

**What we need**: A way to control the behavior of the dependencies within our test. We need an `InventorySystem` that we can configure with any stock level we want, and that doesn't affect other tests. We need a `PaymentGateway` that doesn't actually make network calls but simply tells us if it was called correctly.

### The Solution: Fakes and Mocks

We will create "Fake" objects—simplified, in-memory implementations of our dependencies that are designed for testing.

```python
# tests/fakes.py (New File)
from src.product import Product

class FakeInventory:
    def __init__(self, stock: dict[str, int]):
        self._stock = stock
        self.stock_reduced_for = None

    def check_stock(self, product: Product, quantity: int) -> bool:
        return self._stock.get(product.name, 0) >= quantity

    def reduce_stock(self, product: Product, quantity: int):
        self._stock[product.name] -= quantity
        self.stock_reduced_for = product.name

class FakePaymentGateway:
    def __init__(self):
        self.charged_amount = 0
        self.charged_card = None

    def charge(self, amount: float, card_details: str) -> bool:
        self.charged_amount = amount
        self.charged_card = card_details
        return True
```

These fakes mimic the interface of the real objects but give us complete control. We can set the initial stock and inspect their state after the test (`charged_amount`, `stock_reduced_for`) to verify interactions.

Now, let's rewrite our test using these fakes.

```python
# tests/test_order_processor.py (Rewritten with Fakes)
import pytest
from src.order_processor import OrderProcessor
from src.product import Product
from .fakes import FakeInventory, FakePaymentGateway

@pytest.fixture
def product():
    return Product("Laptop", 1000.0)

def test_process_order_successfully(product):
    # Arrange
    inventory = FakeInventory(stock={"Laptop": 10})
    payment = FakePaymentGateway()
    processor = OrderProcessor(inventory, payment)
    
    # Act
    processor.process_order(product, quantity=8, card_details="1234-5678")
    
    # Assert
    # 1. Check interactions with dependencies
    assert inventory.stock_reduced_for == "Laptop"
    assert payment.charged_amount == 8000.0
    assert payment.charged_card == "1234-5678"
    
    # 2. Check internal state of the object under test
    assert len(processor.processed_orders) == 1

def test_process_order_fails_when_not_enough_stock(product):
    # Arrange
    inventory = FakeInventory(stock={"Laptop": 5}) # Not enough stock
    payment = FakePaymentGateway()
    processor = OrderProcessor(inventory, payment)
    
    # Act & Assert
    with pytest.raises(ValueError, match="Not enough stock."):
        processor.process_order(product, quantity=8, card_details="1234-5678")
```

These tests are now fast, reliable, and isolated. They test the logic of `OrderProcessor` without touching any external systems. We have successfully unit-tested code with side effects.

However, this introduces a new risk: what if our `FakeInventory` behaves differently from the `RealInventory`? Our unit tests would pass, but the application would fail in production. This is where integration testing comes in.

## Integration Testing Within Your Codebase

## Verifying the Contracts Between Components

Unit tests are essential for verifying the logic of a single component in isolation. We've successfully tested our `OrderProcessor` by replacing its dependencies with fakes.

But this creates a blind spot. Our tests prove that `OrderProcessor` works with `FakeInventory`, but they *don't* prove it works with the `RealInventory`. We are trusting that our fake perfectly mimics the real object's contract (method names, arguments, return values, exceptions). If the real `InventorySystem` changes its `reduce_stock` method to `debit_stock`, our unit tests for `OrderProcessor` would still pass, but the application would break.

**Integration tests** fill this gap. They test the interaction *between* two or more real components to ensure they work together as expected.

### Iteration 5: Testing the Collaboration

Let's write an integration test that uses the real `OrderProcessor` and the real `InventorySystem`. We will still fake the `PaymentGateway`, as we want to avoid external network calls, but we will test the direct integration between the order logic and the inventory logic.

```python
# tests/test_integration.py (New File)
import pytest
from src.order_processor import OrderProcessor
from src.dependencies import InventorySystem
from src.product import Product
from .fakes import FakePaymentGateway # We still fake the external service

@pytest.fixture
def product():
    return Product("Keyboard", 75.0)

def test_order_processing_updates_real_inventory(product):
    """
    An integration test to verify OrderProcessor and InventorySystem work together.
    """
    # Arrange
    # Use the REAL InventorySystem
    inventory = InventorySystem() 
    # Use a FAKE PaymentGateway to avoid external calls
    payment = FakePaymentGateway()
    processor = OrderProcessor(inventory, payment)
    
    # Check initial stock level
    assert inventory.check_stock(product, 20) == True

    # Act
    processor.process_order(product, quantity=15, card_details="valid-card")

    # Assert
    # Verify the side effect on the real inventory object
    assert inventory.check_stock(product, 11) == False # 25 - 15 = 10 left
    assert inventory.check_stock(product, 10) == True
```

This test provides a higher level of confidence. It proves that `OrderProcessor` calls the correct methods on `InventorySystem` and that the state of `InventorySystem` is updated as expected.

### The Test Pyramid: Balancing Unit and Integration Tests

This brings us to the concept of the Test Pyramid.

-   **Unit Tests (Base)**: You should have many of these. They are fast, stable, and precisely locate failures. Our tests using `FakeInventory` and `FakePaymentGateway` are unit tests.
-   **Integration Tests (Middle)**: You should have a moderate number of these. They are slower and more brittle than unit tests but verify that components collaborate correctly. Our `test_order_processing_updates_real_inventory` is an integration test.
-   **End-to-End (E2E) Tests (Top)**: You should have very few of these. They test the entire application stack, often by driving a web browser or a public API. They are very slow, brittle, and expensive to run, but provide the highest confidence that the whole system works.

You should focus most of your effort on unit tests, as they provide the best return on investment for catching bugs quickly. Use integration tests strategically to cover the critical seams between your application's components.

### The Journey: From Problem to Solution

Let's review the path we took to build a robust test suite for our synchronous code.

| Iteration | Problem / Failure Mode                               | Technique Applied                               | Result                                                |
| :-------- | :--------------------------------------------------- | :---------------------------------------------- | :---------------------------------------------------- |
| 0         | Testing a simple, pure function.                     | Basic `assert` statement.                       | Confidence in a single algorithm.                     |
| 1         | Logic moved into a class method.                     | Instantiate the class in the test.              | Test adapted to object-oriented structure.            |
| 2         | Shared class instance causes test interdependence.   | `pytest.fixture` for test isolation.            | Each test gets a fresh instance; tests are reliable.  |
| 3         | Temptation to test private methods.                  | Philosophy: Test public behavior, not implementation. | A robust test suite that isn't brittle to refactoring. |
| 4         | Side effects (database, API) make tests slow/unreliable. | Test Doubles (Fakes) and Dependency Injection.  | Fast, isolated unit tests for complex logic.          |
| 5         | Unit tests don't verify component contracts.         | Integration tests for key component collaborations. | Confidence that components work together correctly.   |

### Final Implementation

Here is the final state of our `OrderProcessor` and a selection of the tests we built, representing a healthy mix of unit and integration testing.

```python
# src/order_processor.py (Final)
import uuid
from .product import Product
from .dependencies import InventorySystem, PaymentGateway

class OrderProcessor:
    def __init__(self, inventory: InventorySystem, payment: PaymentGateway):
        self.inventory = inventory
        self.payment = payment
        self.processed_orders = []

    def _generate_order_id(self) -> str:
        return f"ORD-{uuid.uuid4()}"

    def process_order(self, product: Product, quantity: int, card_details: str):
        if not self.inventory.check_stock(product, quantity):
            raise ValueError("Not enough stock.")
        
        total_price = product.price * quantity
        if not self.payment.charge(total_price, card_details):
            raise ValueError("Payment failed.")
            
        self.inventory.reduce_stock(product, quantity)
        
        order_id = self._generate_order_id()
        self.processed_orders.append(order_id)
        return order_id

# tests/test_order_processor.py (Final Unit Test)
def test_process_order_successfully(product):
    # Arrange
    inventory = FakeInventory(stock={"Laptop": 10})
    payment = FakePaymentGateway()
    processor = OrderProcessor(inventory, payment)
    
    # Act
    processor.process_order(product, quantity=8, card_details="1234-5678")
    
    # Assert
    assert inventory.stock_reduced_for == "Laptop"
    assert payment.charged_amount == 8000.0
    assert len(processor.processed_orders) == 1

# tests/test_integration.py (Final Integration Test)
def test_order_processing_updates_real_inventory(product):
    # Arrange
    inventory = InventorySystem() 
    payment = FakePaymentGateway()
    processor = OrderProcessor(inventory, payment)
    
    # Act
    processor.process_order(product, quantity=15, card_details="valid-card")

    # Assert
    assert inventory.check_stock(product, 10) == True
```

### Decision Framework: Unit vs. Integration Test

When should you write which type of test?

| Characteristic      | Choose a **Unit Test** When...                               | Choose an **Integration Test** When...                        |
| :------------------ | :----------------------------------------------------------- | :------------------------------------------------------------ |
| **Goal**            | You want to verify the internal logic of a single component. | You want to verify the contract/collaboration between components. |
| **Dependencies**    | All external dependencies (DB, API, other classes) are replaced with Test Doubles. | One or more dependencies are real, concrete objects.          |
| **Speed**           | You need the test to be extremely fast (milliseconds).       | You can tolerate a slower test (tens or hundreds of ms).      |
| **Scope**           | The test focuses on edge cases, business rules, and algorithms within one class. | The test focuses on data flow and side effects across classes. |
| **Failure Insight** | A failure precisely pinpoints the bug in a specific class.   | A failure indicates a problem in the interaction, requiring more debugging. |
