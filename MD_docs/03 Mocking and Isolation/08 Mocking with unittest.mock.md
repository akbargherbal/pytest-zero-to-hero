# Chapter 8: Mocking with unittest.mock

## Why Mock?

## The Problem: Testing in Isolation

So far, our tests have been self-contained. We've tested functions that take data, transform it, and return a result. This is the ideal scenario for testing. But real-world applications are rarely so simple. They are complex systems that interact with databases, file systems, third-party APIs, and other services.

Consider a function that processes a customer's order. Its job might involve:
1.  Charging the customer's credit card via a payment gateway API.
2.  Saving the order details to a database.
3.  Sending a confirmation email via an email service.

How would you write a unit test for this function? If your test calls the *real* payment gateway, you have several major problems:

-   **Cost:** You might be charged real money for every test run.
-   **Slowness:** Network requests to external services are orders of magnitude slower than in-memory function calls. A test suite with hundreds of such tests could take minutes or hours to run.
-   **Unreliability:** The external service could be down, or your network connection could be flaky. This would cause your tests to fail for reasons completely unrelated to your code's correctness.
-   **Side Effects:** You would be creating fake orders in the database and sending real emails to non-existent addresses with every test run.
-   **Testing Edge Cases is Hard:** How do you test what happens when the payment gateway returns a "Card Declined" error? Or a "Gateway Timeout" error? You can't easily force a real-world service to produce these specific error conditions on demand.

The goal of a **unit test** is to test a single *unit* of code in **isolation**. To do this, we need a way to pretend these external dependencies exist, controlling their behavior so we can test our code's logic reliably and quickly. This is where mocking comes in.

### Phase 1: Establish the Reference Implementation

Let's build a concrete example that demonstrates these problems. We'll create a simple e-commerce order processing system. It has two external dependencies: a `PaymentGateway` and a `NotificationService`.

Our anchor example will be the `process_order` function. We will spend this chapter refining the tests for this single function.

First, let's define the external services. We'll simulate their slowness and potential for failure.

```python
# src/services.py

import time
import random

class PaymentGateway:
    """A simulated external payment gateway."""
    def charge(self, card_details: str, amount: float) -> str:
        """
        Charges the card. Returns a transaction ID on success.
        In a real system, this would make a network request.
        """
        print("\nConnecting to payment gateway...")
        if not card_details or amount <= 0:
            raise ValueError("Invalid card details or amount.")
        
        # Simulate network latency
        time.sleep(2)
        
        # Simulate a chance of failure
        if random.random() < 0.2: # 20% chance of failure
            print("The payment was declined.")
            return "" # Return empty string for a declined payment
        
        transaction_id = f"tx_{random.randint(1000, 9999)}"
        print(f"Charge successful. Transaction ID: {transaction_id}")
        return transaction_id

class NotificationService:
    """A simulated external notification service."""
    def send_receipt(self, email: str, transaction_id: str):
        """Sends a receipt to the customer's email."""
        print(f"\nSending receipt for {transaction_id} to {email}...")
        # Simulate network latency
        time.sleep(1)
        print("Receipt sent.")
        return True
```

Now, here is the function we want to test. It coordinates these two services.

```python
# src/orders.py

from .services import PaymentGateway, NotificationService

def process_order(order_id: str, customer_email: str, card_details: str, amount: float):
    """
    Processes a customer order.
    1. Charges the customer's card.
    2. Sends a receipt if the charge is successful.
    """
    gateway = PaymentGateway()
    notifier = NotificationService()

    print(f"\nProcessing order {order_id}...")
    transaction_id = gateway.charge(card_details, amount)

    if transaction_id:
        print(f"Payment successful for order {order_id}.")
        notifier.send_receipt(customer_email, transaction_id)
        return "Order processed successfully."
    else:
        print(f"Payment failed for order {order_id}.")
        return "Payment failed."
```

Finally, let's write our first, naive test. This is an **integration test**, not a unit test, because it uses the real, live `services`.

```python
# tests/test_orders_naive.py

from src.orders import process_order

def test_process_order_success_naive():
    """
    A slow and unreliable test for a successful order.
    This test will take ~3 seconds to run.
    """
    result = process_order(
        order_id="order_123",
        customer_email="test@example.com",
        card_details="tok_valid",
        amount=49.99
    )
    assert result == "Order processed successfully."
```

Let's run this test and see the problems firsthand.

```bash
$ pytest -v -s tests/test_orders_naive.py
# The -s flag is used to show the print statements

=========================== test session starts ============================
...
collected 1 item

tests/test_orders_naive.py::test_process_order_success_naive 
Processing order order_123...

Connecting to payment gateway...
Charge successful. Transaction ID: tx_...
Payment successful for order order_123.

Sending receipt for tx_... to test@example.com...
Receipt sent.
PASSED [100%]

======================= 1 passed in 3.05s ========================
```

The test passed, but look at the execution time: **3.05 seconds**. For one test! A suite of 100 such tests would take over 5 minutes. Furthermore, if we run it enough times, it will eventually fail due to the `random.random()` check we added to simulate unreliability.

This is the core problem that mocking solves. We need to test the logic of `process_order` *without* actually calling the slow, unreliable, and costly external services.

## What Is a Mock?

## Test Doubles, Stubs, and Mocks

To test our code in isolation, we replace real dependency objects (like an instance of `PaymentGateway`) with fake objects that we control. These fake objects are generically called **Test Doubles**. The term comes from the idea of a "stunt double" in movies—a stand-in that looks and acts like the real thing for a specific, controlled scene.

There are several types of test doubles, but the most common one we'll use is a **Mock**.

A **Mock Object** is a smart test double that:
1.  **Stands in** for a real object.
2.  **Can be configured** to return specific values from its methods.
3.  **Records how it was used**, allowing you to make assertions about which methods were called, how many times, and with what arguments.

In Python, the standard library for creating and managing mocks is `unittest.mock`. Even though the name includes "unittest" (the built-in test framework), it is a general-purpose library that works perfectly with pytest.

The core idea is to replace this:

`Our Code -> Real Payment Gateway`

with this:

`Our Code -> Mock Payment Gateway <- Our Test`

Our test code will:
1.  Create a mock object that looks and feels like a `PaymentGateway`.
2.  Tell the `process_order` function to use our mock instead of the real one.
3.  Configure the mock to behave in a specific way (e.g., "when `charge` is called, return `tx_12345`").
4.  Run `process_order`.
5.  Ask the mock questions to verify the interaction (e.g., "were you called? Was your `charge` method called with an amount of 49.99?").

This approach solves all our previous problems:
-   **Cost:** No real services are called. It's free.
-   **Slowness:** Mocks are just in-memory Python objects. They are incredibly fast.
-   **Unreliability:** Mocks are 100% deterministic. They do exactly what you tell them to do.
-   **Side Effects:** No databases are touched, no emails are sent.
-   **Testing Edge Cases:** We can easily configure a mock to simulate a "Card Declined" error or any other scenario we want to test.

## Creating Mocks with Mock()

## The `Mock` Class

The `unittest.mock` module provides a class called `Mock` (and a more powerful subclass `MagicMock`) that serves as a generic mock object. Let's see it in action.

```python
# You can run this in a Python interpreter
from unittest.mock import Mock

# Create a mock object
mock_gateway = Mock()

# You can call any method on it, and it will return another Mock
result = mock_gateway.charge("details", 100)
print(result)
# Output: <Mock name='mock.charge()' id='...'>

# You can access any attribute, and it will also return a Mock
api_key = mock_gateway.api_key
print(api_key)
# Output: <Mock name='mock.api_key' id='...'>
```

This is the default behavior. A `Mock` object is a blank slate that will accept any operation you perform on it and record that it happened. This is useful, but to make it behave like our `PaymentGateway`, we need to configure it.

We can pre-configure a mock to have specific attributes or methods with return values.

```python
from unittest.mock import Mock

# Configure a mock to simulate a successful charge
mock_gateway = Mock()
mock_gateway.charge.return_value = "tx_success_123"

# Now when we call charge, it returns our configured value
transaction_id = mock_gateway.charge(card_details="tok_valid", amount=50.00)
print(transaction_id)
# Output: tx_success_123

# We can also check if the method was called
mock_gateway.charge.assert_called()

# And check *what* it was called with
mock_gateway.charge.assert_called_with(card_details="tok_valid", amount=50.00)
```

This is the fundamental mechanism. However, creating a mock is one thing; getting our `process_order` function to *use* it is another. The `process_order` function creates its own `PaymentGateway` instance internally. We need a way to intercept that creation and substitute our mock. This is called **patching**.

## Patching Functions with @patch

## Intercepting Code with `@patch`

The most common tool you'll use from `unittest.mock` is `patch`. It's a powerful decorator (or context manager) that temporarily replaces an object in a specific module with a mock.

The key to using `patch` correctly is to provide the path to the object **where it is looked up**, not where it is defined. In our case, the `process_order` function lives in `src.orders` and it does `from .services import PaymentGateway`. Therefore, inside `src.orders`, `PaymentGateway` refers to `src.orders.PaymentGateway`. This is the target we must patch.

### Iteration 1: Replacing the Slow Services

Let's write a new test that uses `@patch` to replace both `PaymentGateway` and `NotificationService`.

**The Goal:** Run the test for `process_order` without the 3-second delay.

Here is our first attempt.

```python
# tests/test_orders_mocked.py

from unittest.mock import patch
from src.orders import process_order

# The target string is 'module.submodule.ClassName'
# Note the order: decorators are applied from the bottom up.
# The first argument to the test function corresponds to the *inner-most* decorator.
# So, mock_notifier comes first, then mock_gateway.
@patch('src.orders.PaymentGateway')
@patch('src.orders.NotificationService')
def test_process_order_success_mocked(mock_notifier_class, mock_gateway_class):
    # The patch gives us mock *classes*, not instances.
    # We can configure the behavior of instances that will be created from these classes.
    mock_gateway_instance = mock_gateway_class.return_value
    mock_gateway_instance.charge.return_value = "tx_mock_456"

    mock_notifier_instance = mock_notifier_class.return_value

    # Run the function under test
    result = process_order(
        order_id="order_456",
        customer_email="mock@example.com",
        card_details="tok_mock",
        amount=19.99
    )

    # Assert the final result
    assert result == "Order processed successfully."
```

Let's run this new test.

```bash
$ pytest -v -s tests/test_orders_mocked.py

=========================== test session starts ============================
...
collected 1 item

tests/test_orders_mocked.py::test_process_order_success_mocked 
Processing order order_456...
Payment successful for order order_456.
PASSED [100%]

======================= 1 passed in 0.02s ========================
```

Look at that! The test passed in **0.02 seconds**. We have successfully replaced the slow external services with fast mock objects.

However, our test is incomplete. It asserts that `process_order` returns the correct string, but it doesn't verify that the services were used correctly. Did we actually call `gateway.charge` with the right amount? Did we call `notifier.send_receipt` with the correct email and transaction ID? Right now, we have no idea. A test that passes for the wrong reasons is just as dangerous as a test that fails. We need to add assertions about the *interactions* with our mocks.

## Common Mock Assertions (assert_called, assert_called_with)

## Verifying Mock Interactions

Mocks are not just stand-ins; they are spies. They record every interaction you have with them. After running your code, you can query the mock to ensure it was used as you expected.

The most common assertion methods are:
-   `mock_object.method.assert_called()`: Asserts that the method was called at least once.
-   `mock_object.method.assert_called_once()`: Asserts that the method was called exactly once.
-   `mock_object.method.assert_called_with(*args, **kwargs)`: Asserts that the *last* call to the method was with these specific arguments.
-   `mock_object.method.assert_called_once_with(*args, **kwargs)`: A combination of the above. Asserts it was called exactly once, and that one call was with these arguments.
-   `mock_object.method.assert_not_called()`: Asserts the method was never called.
-   `mock_object.method.call_count`: An integer property that tells you how many times the method was called.

### Iteration 2: Verifying the Gateway and Notifier Calls

Let's improve our previous test by adding assertions to verify that the gateway and notifier were called correctly.

**Current Limitation:** Our test only checks the final return value. It doesn't confirm that the dependencies were used correctly.

**New Scenario:** We need to ensure `gateway.charge` is called with the correct `card_details` and `amount`, and that `notifier.send_receipt` is called with the correct `email` and the `transaction_id` returned by the gateway.

Let's add these assertions to our test. To demonstrate how the assertions work, we'll start by making a mistake on purpose. Let's assert that the `amount` was `29.99` when it was actually `19.99`.

```python
# tests/test_orders_mocked.py (updated test)

from unittest.mock import patch
from src.orders import process_order

@patch('src.orders.PaymentGateway')
@patch('src.orders.NotificationService')
def test_process_order_success_with_assertions(mock_notifier_class, mock_gateway_class):
    # Arrange: Configure the mocks
    mock_gateway_instance = mock_gateway_class.return_value
    mock_gateway_instance.charge.return_value = "tx_mock_456"

    mock_notifier_instance = mock_notifier_class.return_value

    # Act: Run the function under test
    result = process_order(
        order_id="order_456",
        customer_email="mock@example.com",
        card_details="tok_mock",
        amount=19.99
    )

    # Assert: Check the final result and the interactions
    assert result == "Order processed successfully."

    # Verify the gateway was used correctly
    mock_gateway_instance.charge.assert_called_once_with("tok_mock", 29.99) # INTENTIONAL ERROR

    # Verify the notifier was used correctly
    mock_notifier_instance.send_receipt.assert_called_once_with(
        "mock@example.com", "tx_mock_456"
    )
```

Now, let's run this and see the failure.

```bash
$ pytest -v tests/test_orders_mocked.py::test_process_order_success_with_assertions
```

### Diagnostic Analysis: Reading the Failure

**The complete output**:
```
=========================== test session starts ============================
...
collected 1 item

tests/test_orders_mocked.py::test_process_order_success_with_assertions FAILED [100%]

================================= FAILURES =================================
___ test_process_order_success_with_assertions ___

mock_notifier_class = <MagicMock name='NotificationService' id='...'>
mock_gateway_class = <MagicMock name='PaymentGateway' id='...'>

    @patch('src.orders.PaymentGateway')
    @patch('src.orders.NotificationService')
    def test_process_order_success_with_assertions(mock_notifier_class, mock_gateway_class):
        # ... (setup code) ...
    
        # Verify the gateway was used correctly
>       mock_gateway_instance.charge.assert_called_once_with("tok_mock", 29.99)
E       AssertionError: expected call not found.
E       Expected: charge('tok_mock', 29.99)
E       Actual: charge('tok_mock', 19.99)

tests/test_orders_mocked.py:28: AssertionError
========================= short test summary info ==========================
FAILED tests/test_orders_mocked.py::test_process_order_success_with_assertions - AssertionError: expected call not found.
============================ 1 failed in 0.05s =============================
```

**Let's parse this section by section**:

1.  **The summary line**: `FAILED ... - AssertionError: expected call not found.`
    -   What this tells us: The test failed because a mock assertion failed. Specifically, we asserted a method was called in a certain way, but it wasn't.

2.  **The traceback**:
    ```python
    >       mock_gateway_instance.charge.assert_called_once_with("tok_mock", 29.99)
    E       AssertionError: expected call not found.
    ```
    -   What this tells us: The exact line that failed was our call to `assert_called_once_with`.

3.  **The assertion introspection**:
    ```
    E       Expected: charge('tok_mock', 29.99)
    E       Actual: charge('tok_mock', 19.99)
    ```
    -   What this tells us: This is the most valuable part. The `unittest.mock` library provides an incredibly clear report. We expected a call with the amount `29.99`, but the actual call was made with `19.99`.

**Root cause identified**: Our assertion for the `amount` argument was incorrect.
**Why the current approach can't solve this**: The test itself is fine; the *assertion data* is wrong. This failure proves our test is working correctly—it caught a discrepancy.
**What we need**: To fix the test, we simply need to update the assertion to reflect the correct, expected arguments.

### The Solution

Let's fix the assertion and rerun the test.

```python
# tests/test_orders_mocked.py (fixed test)

from unittest.mock import patch
from src.orders import process_order

@patch('src.orders.PaymentGateway')
@patch('src.orders.NotificationService')
def test_process_order_success_with_assertions_fixed(mock_notifier_class, mock_gateway_class):
    # Arrange
    mock_gateway_instance = mock_gateway_class.return_value
    mock_gateway_instance.charge.return_value = "tx_mock_456"
    mock_notifier_instance = mock_notifier_class.return_value

    # Act
    result = process_order(
        order_id="order_456",
        customer_email="mock@example.com",
        card_details="tok_mock",
        amount=19.99
    )

    # Assert
    assert result == "Order processed successfully."
    mock_gateway_instance.charge.assert_called_once_with("tok_mock", 19.99) # CORRECTED
    mock_notifier_instance.send_receipt.assert_called_once_with(
        "mock@example.com", "tx_mock_456"
    )
```

Running this test now passes, and we have much higher confidence in its correctness. We've verified the final output *and* the internal interactions with its dependencies.

## Mock Side Effects and Return Values

## Controlling Mock Behavior

So far, we've used `return_value` to specify what a mock method should return. This is the most common way to configure a mock. But what if we need to simulate more complex behavior, like an error?

### Iteration 3: Testing Failure Paths

**Current Limitation:** We've only tested the "happy path" where the payment succeeds. A robust test suite must also cover failure scenarios.

**New Scenario:** What happens if the payment gateway declines the charge? Our `process_order` function should return "Payment failed." and should *not* call the notification service.

To test this, we need to configure our mock gateway's `charge` method to simulate a failure. In our real `PaymentGateway`, a failed charge returns an empty string. Let's simulate that.

We can also use another powerful attribute: `side_effect`. The `side_effect` argument can be used to raise exceptions or to return different values on subsequent calls.

-   `mock.method.side_effect = Exception("Boom!")`: Calling `mock.method()` will now raise `Exception("Boom!")`.
-   `mock.method.side_effect = [val1, val2, val3]`: The first call will return `val1`, the second `val2`, etc.

Let's write two new tests: one for a declined payment and one for a gateway exception.

```python
# tests/test_orders_mocked.py (new tests for failure paths)

from unittest.mock import patch
import pytest
from src.orders import process_order

# We'll need a custom exception for one of our tests
class GatewayError(Exception):
    pass

@patch('src.orders.PaymentGateway')
@patch('src.orders.NotificationService')
def test_process_order_payment_declined(mock_notifier_class, mock_gateway_class):
    # Arrange: Configure the gateway to simulate a declined payment (returns empty string)
    mock_gateway_instance = mock_gateway_class.return_value
    mock_gateway_instance.charge.return_value = "" # Simulate failure
    mock_notifier_instance = mock_notifier_class.return_value

    # Act
    result = process_order("order_789", "fail@example.com", "tok_declined", 100.00)

    # Assert
    assert result == "Payment failed."
    mock_gateway_instance.charge.assert_called_once_with("tok_declined", 100.00)
    
    # Crucially, assert the notifier was *not* called
    mock_notifier_instance.send_receipt.assert_not_called()

@patch('src.orders.PaymentGateway')
@patch('src.orders.NotificationService')
def test_process_order_gateway_error(mock_notifier_class, mock_gateway_class):
    # Arrange: Configure the gateway to raise an exception
    mock_gateway_instance = mock_gateway_class.return_value
    mock_gateway_instance.charge.side_effect = GatewayError("Connection timed out")
    mock_notifier_instance = mock_notifier_class.return_value

    # Act & Assert: Use pytest.raises to check for the exception
    with pytest.raises(GatewayError, match="Connection timed out"):
        process_order("order_999", "error@example.com", "tok_error", 200.00)

    # Assert that the notifier was not called in this case either
    mock_notifier_instance.send_receipt.assert_not_called()
```

These tests now pass and give us confidence that our `process_order` function correctly handles two critical failure modes. Mocks made this trivial to test; trying to test this with a real payment gateway would be nearly impossible.

## Combining Mocks and Fixtures

## The Pytest Way: Mocks as Fixtures

Using multiple `@patch` decorators on every test function works, but it has some drawbacks:
-   **Verbosity:** The decorators stack up, adding boilerplate to each test.
-   **Repetition:** If many tests need the same mock setup, you're repeating the `@patch` lines everywhere.
-   **Argument Ordering:** You have to remember the reverse order of arguments (`@patch('A')`, `@patch('B')` means the test signature is `def test(mock_B, mock_A):`). This is a common source of bugs.

Pytest's fixture system provides a much cleaner and more maintainable way to manage mocks. We can move the patching logic inside a fixture. The `patch` object can also be used as a context manager, which is perfect for fixtures.

### Iteration 4: Refactoring to Fixtures

**Current Limitation:** Our tests are becoming repetitive with multiple `@patch` decorators.

**Goal:** Encapsulate the mock setup into reusable fixtures to make tests cleaner and more declarative.

Let's create two fixtures, `mock_gateway` and `mock_notifier`, in our test file.

```python
# tests/test_orders_fixtures.py

import pytest
from unittest.mock import patch
from src.orders import process_order

@pytest.fixture
def mock_gateway():
    """Fixture to mock the PaymentGateway."""
    with patch('src.orders.PaymentGateway') as mock_gateway_class:
        mock_instance = mock_gateway_class.return_value
        yield mock_instance

@pytest.fixture
def mock_notifier():
    """Fixture to mock the NotificationService."""
    with patch('src.orders.NotificationService') as mock_notifier_class:
        mock_instance = mock_notifier_class.return_value
        yield mock_instance

def test_process_order_success_with_fixtures(mock_gateway, mock_notifier):
    # Arrange
    mock_gateway.charge.return_value = "tx_fixture_123"

    # Act
    result = process_order(
        order_id="order_fixture",
        customer_email="fixture@example.com",
        card_details="tok_fixture",
        amount=50.00
    )

    # Assert
    assert result == "Order processed successfully."
    mock_gateway.charge.assert_called_once_with("tok_fixture", 50.00)
    mock_notifier.send_receipt.assert_called_once_with(
        "fixture@example.com", "tx_fixture_123"
    )

def test_process_order_payment_declined_with_fixtures(mock_gateway, mock_notifier):
    # Arrange
    mock_gateway.charge.return_value = "" # Simulate failure

    # Act
    result = process_order("order_fail", "fail_fixture@example.com", "tok_declined", 75.00)

    # Assert
    assert result == "Payment failed."
    mock_gateway.charge.assert_called_once_with("tok_declined", 75.00)
    mock_notifier.send_receipt.assert_not_called()
```

This is a significant improvement.
-   **Declarative:** The test function now clearly states its dependencies: `mock_gateway` and `mock_notifier`.
-   **Clean:** The test body contains only the logic relevant to that specific test case (Arrange, Act, Assert). The boilerplate of patching is hidden in the fixtures.
-   **Reusable:** These fixtures can be used by any test in the file. If you move them to a `conftest.py` file, they can be used by your entire test suite.

This pattern of wrapping `patch` in a fixture is the idiomatic way to use `unittest.mock` with pytest.

### Synthesis: The Complete Journey

Let's review our progress. We started with a slow, unreliable integration test and progressively refined it into a fast, robust, and maintainable suite of unit tests.

| Iteration | Failure Mode / Limitation                               | Technique Applied                               | Result                                                              |
| :-------- | :------------------------------------------------------ | :---------------------------------------------- | :------------------------------------------------------------------ |
| 0         | Test is slow (~3s) and unreliable.                      | None (direct call to real services)             | An integration test, not a unit test.                               |
| 1         | Slowness and unreliability.                             | `@patch` decorator                              | Test is fast (~0.02s) but doesn't verify interactions.                |
| 2         | Test passes but doesn't prove correctness.              | `assert_called_with`                            | Test now verifies that dependencies are called with correct arguments. |
| 3         | Only the "happy path" is tested.                        | `return_value` and `side_effect` for failures   | Test suite now covers success, decline, and exception scenarios.     |
| 4         | Test setup is verbose and repetitive (`@patch` stack).  | `patch` as a context manager inside fixtures    | Tests are clean, declarative, and setup logic is reusable.          |

### Final Implementation

Here is what a final, production-ready test file for our `process_order` function might look like, using all the techniques we've learned.

```python
# tests/final/test_orders.py

import pytest
from unittest.mock import patch
from src.orders import process_order

# A custom exception for testing error paths
class GatewayError(Exception):
    pass

@pytest.fixture
def mock_gateway():
    """Fixture to mock the PaymentGateway, returning the instance."""
    # Using autospec=True is a best practice. It ensures the mock has the same
    # API as the real object. If you try to call a non-existent method,
    # your test will fail.
    with patch('src.orders.PaymentGateway', autospec=True) as mock_gateway_class:
        mock_instance = mock_gateway_class.return_value
        yield mock_instance

@pytest.fixture
def mock_notifier():
    """Fixture to mock the NotificationService, returning the instance."""
    with patch('src.orders.NotificationService', autospec=True) as mock_notifier_class:
        mock_instance = mock_notifier_class.return_value
        yield mock_instance

def test_process_order_success(mock_gateway, mock_notifier):
    """Tests the successful order processing path."""
    # Arrange
    mock_gateway.charge.return_value = "tx_final_123"

    # Act
    result = process_order("order_final", "final@example.com", "tok_final", 99.99)

    # Assert
    assert result == "Order processed successfully."
    mock_gateway.charge.assert_called_once_with("tok_final", 99.99)
    mock_notifier.send_receipt.assert_called_once_with("final@example.com", "tx_final_123")

def test_process_order_payment_declined(mock_gateway, mock_notifier):
    """Tests the path where payment is declined."""
    # Arrange
    mock_gateway.charge.return_value = ""

    # Act
    result = process_order("order_declined", "declined@example.com", "tok_declined", 150.00)

    # Assert
    assert result == "Payment failed."
    mock_gateway.charge.assert_called_once_with("tok_declined", 150.00)
    mock_notifier.send_receipt.assert_not_called()

def test_process_order_gateway_exception(mock_gateway, mock_notifier):
    """Tests that an exception from the gateway is propagated."""
    # Arrange
    mock_gateway.charge.side_effect = GatewayError("Gateway is down")

    # Act & Assert
    with pytest.raises(GatewayError, match="Gateway is down"):
        process_order("order_error", "error@example.com", "tok_error", 250.00)
    
    mock_notifier.send_receipt.assert_not_called()
```
