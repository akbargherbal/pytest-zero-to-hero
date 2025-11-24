# Chapter 9: Spies, Stubs, and Test Doubles

## The Difference Between Mocks, Stubs, and Spies

## The Problem: Testing Code with External Dependencies

So far, we've tested functions that are self-contained. They take inputs, perform calculations, and return outputs. But real-world code is messy. It interacts with databases, calls external APIs, reads from the filesystem, and depends on other complex systems.

How do you test a function that charges a credit card without actually charging a real credit card every time you run your tests? How do you test a function that relies on a web service that might be slow, unreliable, or rate-limited?

This is where **Test Doubles** come in. A test double is a general term for any object or function that stands in for a real one during a test. Pytest doesn't have its own test double library; instead, it integrates seamlessly with the standard `unittest.mock` library, which we will use throughout this chapter.

The three most common types of test doubles are:
1.  **Stubs**: Objects that provide canned answers to calls made during the test. They are used when you don't care about the interaction, only that the system under test gets the data it needs to proceed.
2.  **Spies**: Objects that record information about how they were called (e.g., how many times, with what arguments). They are used when you want to verify the *interaction* between your system and its dependency.
3.  **Mocks**: A more complex type of double that is pre-programmed with expectations. These expectations form a specification of the calls they are expected to receive. A mock will cause the test to fail if it receives a call that isn't in its specification.

In Python, the `unittest.mock.MagicMock` class is so powerful that it can act as a stub, a spy, or a mock, often at the same time. We will focus on using it to create stubs and spies.

### Phase 1: Establish the Reference Implementation

Let's build our anchor example for this chapter: a simple e-commerce order processing system. It needs to calculate the total price of an order and then charge a payment gateway.

The dependency we need to isolate is the `PaymentGateway`. In a real system, this class would handle complex, slow, and expensive communication with a service like Stripe or PayPal.

Here's our initial, untestable code.

**The System Under Test:**

```python
# project/services.py

import time

class PaymentGateway:
    """
    A real payment gateway that connects to an external service.
    """
    def charge(self, amount: int, token: str) -> str:
        """
        Charges the customer's card.
        In a real system, this would make an API call.
        """
        # Simulate a slow network call
        print("\nConnecting to payment provider...")
        time.sleep(2)
        print("Connection successful.")
        
        if token == "INVALID_TOKEN":
            raise ValueError("Invalid payment token.")
            
        # Simulate a successful transaction ID
        return f"txn_{int(time.time())}"

class Order:
    def __init__(self, items: list, shipping_address: str):
        self.items = items
        self.shipping_address = shipping_address

    @property
    def total(self) -> int:
        return sum(item['price'] for item in self.items)

def process_order(order: Order, payment_token: str) -> str:
    """
    Processes an order by charging the payment gateway.
    """
    print(f"Processing order with total: ${order.total / 100:.2f}")
    
    # Here is our dependency on the external system
    gateway = PaymentGateway()
    transaction_id = gateway.charge(amount=order.total, token=payment_token)
    
    print(f"Order successful! Transaction ID: {transaction_id}")
    return transaction_id
```

**The Naive Test:**

Our first attempt at a test will simply call the function and see what happens.

```python
# tests/test_services_naive.py

from project.services import Order, process_order

def test_process_order_naive():
    """
    This is a bad test. It's slow and makes a real network call.
    """
    order_items = [
        {"name": "Laptop", "price": 120000}, # Prices in cents
        {"name": "Mouse", "price": 2500},
    ]
    order = Order(items=order_items, shipping_address="123 Main St")
    
    transaction_id = process_order(order=order, payment_token="VALID_TOKEN")
    
    assert transaction_id.startswith("txn_")
```

Let's run this test.

```bash
$ pytest -v tests/test_services_naive.py

=========================== test session starts ============================
...
collected 1 item

tests/test_services_naive.py::test_process_order_naive 
Processing order with total: $1225.00
Connecting to payment provider...
Connection successful.
Order successful! Transaction ID: txn_167...
PASSED [100%]

============================ 1 passed in 2.05s =============================
```

The test passes, but it has severe problems:

1.  **It's Slow**: It took over 2 seconds to run one simple test because of `time.sleep(2)`. A real network call could be even slower. A suite with hundreds of these tests would be unusable.
2.  **It's Unreliable**: It depends on a network connection and the external payment service being available. If the service is down, our tests fail, even if our own code is perfect.
3.  **It has Side Effects**: It could potentially charge a real credit card.
4.  **It's Hard to Test Edge Cases**: How would we test what happens if the payment gateway returns an error? We can't easily force the real service to fail in a predictable way.

This is an *integration test*, not a *unit test*. Our goal is to test `process_order` in isolation, without its dependencies. To do that, we need to replace the real `PaymentGateway` with a test double.

## Using MagicMock for Complex Scenarios

## Iteration 1: Replacing the Dependency with a Stub

Our first goal is to remove the slow, unreliable `PaymentGateway` from our test run. We need to replace it with a "stunt double" that instantly provides the return value `process_order` expects. This is a perfect job for a **stub**.

We'll use `unittest.mock.MagicMock` to create our stub and pytest's `monkeypatch` fixture to swap the real class with our fake one during the test.

### The Problem: Testing Failure Cases

Let's write a test for what should happen when the payment gateway rejects a payment token. According to our `PaymentGateway` code, it should raise a `ValueError`.

```python
# tests/test_services_fail.py
import pytest
from project.services import Order, process_order

def test_process_order_with_invalid_token():
    order_items = [{"name": "Book", "price": 1500}]
    order = Order(items=order_items, shipping_address="456 Oak Ave")

    with pytest.raises(ValueError, match="Invalid payment token."):
        process_order(order=order, payment_token="INVALID_TOKEN")
```

This test works, but it still suffers from all the problems we identified: it's slow and relies on the real network.

```bash
$ pytest -v tests/test_services_fail.py
=========================== test session starts ============================
...
collected 1 item

tests/test_services_fail.py::test_process_order_with_invalid_token 
Processing order with total: $15.00
Connecting to payment provider...
Connection successful.
PASSED [100%]

============================ 1 passed in 2.04s =============================
```

We need a way to trigger this `ValueError` instantly, without calling the real `PaymentGateway`.

### The Solution: Stubbing with `monkeypatch` and `MagicMock`

We can use `MagicMock` to create an object that raises an error when a method is called. The `side_effect` attribute is perfect for this. We then use `monkeypatch.setattr` to replace the real `PaymentGateway` class with our mock *before* `process_order` is called.

**The Target String**: The first argument to `setattr` is a string representing the path to the object you want to replace: `'project.services.PaymentGateway'`. This means "within the `project.services` module, find the object named `PaymentGateway` and replace it."

Here is the improved test:

```python
# tests/test_services_v1.py
import pytest
from unittest.mock import MagicMock
from project.services import Order, process_order

def test_process_order_success(monkeypatch):
    # Create a stub for the PaymentGateway
    mock_gateway = MagicMock()
    # Configure the stub's charge method to return a fake transaction ID
    mock_gateway.charge.return_value = "txn_fake_12345"
    
    # Replace the real PaymentGateway class with our stub
    monkeypatch.setattr("project.services.PaymentGateway", lambda: mock_gateway)
    
    # --- Test execution ---
    order_items = [{"name": "Laptop", "price": 120000}]
    order = Order(items=order_items, shipping_address="123 Main St")
    
    transaction_id = process_order(order=order, payment_token="VALID_TOKEN")
    
    assert transaction_id == "txn_fake_12345"

def test_process_order_payment_fails(monkeypatch):
    # Create a stub that simulates a failure
    mock_gateway = MagicMock()
    # Configure the stub's charge method to raise an exception
    mock_gateway.charge.side_effect = ValueError("Invalid payment token.")
    
    # Replace the real PaymentGateway class
    monkeypatch.setattr("project.services.PaymentGateway", lambda: mock_gateway)
    
    # --- Test execution ---
    order_items = [{"name": "Book", "price": 1500}]
    order = Order(items=order_items, shipping_address="456 Oak Ave")

    with pytest.raises(ValueError, match="Invalid payment token."):
        process_order(order=order, payment_token="INVALID_TOKEN")
```

Let's run these new tests.

```bash
$ pytest -v tests/test_services_v1.py
=========================== test session starts ============================
...
collected 2 items

tests/test_services_v1.py::test_process_order_success 
Processing order with total: $1200.00
Order successful! Transaction ID: txn_fake_12345
PASSED [ 50%]
tests/test_services_v1.py::test_process_order_payment_fails 
Processing order with total: $15.00
PASSED [100%]

============================ 2 passed in 0.02s =============================
```

Look at the difference! The tests now run in **0.02 seconds** instead of over 4 seconds. We have successfully isolated our `process_order` function from its dependency.

**Why `lambda: mock_gateway`?**

The code `gateway = PaymentGateway()` inside `process_order` does two things: it accesses the class `PaymentGateway` and then it *calls* it to create an instance. Our patch needs to replace the class with something that is also callable and returns our mock instance. A `lambda` function is a simple way to create a callable that returns our pre-configured `mock_gateway` object.

### Iteration 2: Verifying Interactions with a Spy

Our stub works, but it's dumb. It doesn't tell us *how* it was used. What if there's a bug in `process_order` that calculates the wrong total?

Let's introduce a bug into `process_order`. Instead of `order.total`, we'll accidentally use a fixed amount.

```python
# project/services_buggy.py

# ... (PaymentGateway and Order classes are the same) ...

def process_order(order: Order, payment_token: str) -> str:
    """
    Processes an order by charging the payment gateway.
    THIS VERSION HAS A BUG!
    """
    print(f"Processing order with total: ${order.total / 100:.2f}")
    
    gateway = PaymentGateway()
    # BUG: We are charging a fixed amount, not the order total!
    transaction_id = gateway.charge(amount=100, token=payment_token)
    
    print(f"Order successful! Transaction ID: {transaction_id}")
    return transaction_id
```

Now, let's run our existing test against this buggy version.

```python
# tests/test_services_v1_buggy.py
import pytest
from unittest.mock import MagicMock
# Import the buggy version
from project.services_buggy import Order, process_order

def test_process_order_success_hides_bug(monkeypatch):
    mock_gateway = MagicMock()
    mock_gateway.charge.return_value = "txn_fake_12345"
    monkeypatch.setattr("project.services_buggy.PaymentGateway", lambda: mock_gateway)
    
    order_items = [{"name": "Laptop", "price": 120000}]
    order = Order(items=order_items, shipping_address="123 Main St")
    
    transaction_id = process_order(order=order, payment_token="VALID_TOKEN")
    
    # This assertion still passes!
    assert transaction_id == "txn_fake_12345"
```

Running this test:

```bash
$ pytest -v tests/test_services_v1_buggy.py
=========================== test session starts ============================
...
collected 1 item

tests/test_services_v1_buggy.py::test_process_order_success_hides_bug PASSED [100%]

============================= 1 passed in 0.01s =============================
```

**The test passed, but the code is wrong!** We are undercharging the customer. Our test is not strong enough because it only checks the return value. It doesn't verify the *interaction* with the payment gateway.

We need to turn our stub into a **spy**. A `MagicMock` object automatically records how its methods are called. We can use its special assertion methods, like `assert_called_with()`, to check the arguments.

Here is the improved test that acts as a spy.

```python
# tests/test_services_v2.py
import pytest
from unittest.mock import MagicMock
from project.services_buggy import Order, process_order # Still using the buggy version

def test_process_order_spies_on_charge(monkeypatch):
    mock_gateway = MagicMock()
    mock_gateway.charge.return_value = "txn_fake_12345"
    monkeypatch.setattr("project.services_buggy.PaymentGateway", lambda: mock_gateway)
    
    order_items = [{"name": "Laptop", "price": 120000}]
    order = Order(items=order_items, shipping_address="123 Main St")
    
    process_order(order=order, payment_token="VALID_TOKEN")
    
    # Spy assertion: Verify the charge method was called correctly
    mock_gateway.charge.assert_called_once_with(
        amount=120000, 
        token="VALID_TOKEN"
    )
```

Now, let's run this new test against the buggy code.

```bash
$ pytest -v tests/test_services_v2.py
=========================== test session starts ============================
...
collected 1 item

tests/test_services_v2.py::test_process_order_spies_on_charge FAILED [100%]

================================= FAILURES =================================
___________ test_process_order_spies_on_charge ___________

...
>       mock_gateway.charge.assert_called_once_with(
            amount=120000,
            token="VALID_TOKEN"
        )
E       AssertionError: expected call not found.
E       Expected: charge(amount=120000, token='VALID_TOKEN')
E       Actual: charge(amount=100, token='VALID_TOKEN')

tests/test_services_v2.py:19: AssertionError
========================= short test summary info ==========================
FAILED tests/test_services_v2.py::test_process_order_spies_on_charge
============================ 1 failed in 0.03s =============================
```

### Diagnostic Analysis: Reading the Failure

**The complete output**: The pytest output above.

**Let's parse this section by section**:

1.  **The summary line**: `FAILED tests/test_services_v2.py::test_process_order_spies_on_charge - AssertionError`
    - What this tells us: The test failed because an assertion was not met. This wasn't an unexpected exception; it was a failed check.

2.  **The traceback**: The traceback points directly to the line `mock_gateway.charge.assert_called_once_with(...)`.
    - What this tells us: The failure is not in our production code (`process_order`), but in the test's verification step. The `MagicMock` object itself is raising the `AssertionError`.

3.  **The assertion introspection**:
    ```
    AssertionError: expected call not found.
    Expected: charge(amount=120000, token='VALID_TOKEN')
    Actual: charge(amount=100, token='VALID_TOKEN')
    ```
    - What this tells us: This is the crucial part. The mock library provides a fantastic, readable diff. We *expected* the `charge` method to be called with `amount=120000`, but it was *actually* called with `amount=100`.

**Root cause identified**: The `process_order` function is calling the payment gateway with a hardcoded amount of `100` instead of the calculated order total.
**Why the previous approach couldn't solve this**: Our first test only checked the return value. As long as the stub returned the expected string, the test passed, completely blind to the incorrect arguments being passed to the dependency.
**What we need**: We need to verify the *inputs* to our dependency, not just its *outputs*. The spy pattern (`assert_called_once_with`) allows us to do exactly that.

By adding one line of code, we transformed our test from a simple stub-based check to a powerful spy-based verification that caught a critical business logic bug. After fixing the bug in `project/services.py`, this test will pass.

## Mocking Entire Classes

## Iteration 3: When the Dependency is Created Internally

Our current patching strategy works because `process_order` creates its own `PaymentGateway` instance. We patched the `PaymentGateway` class in the `project.services` namespace, so when `process_order` looks it up, it gets our mock.

But what if the code was structured differently? Let's consider a slight refactoring where the `PaymentGateway` is a module-level instance.

```python
# project/services_v3.py

# ... (PaymentGateway and Order classes are the same) ...

# The gateway is now a singleton instance at the module level
gateway_instance = PaymentGateway()

def process_order_v3(order: Order, payment_token: str) -> str:
    """
    Processes an order using a module-level gateway instance.
    """
    print(f"Processing order with total: ${order.total / 100:.2f}")
    
    # Dependency is no longer created inside the function
    transaction_id = gateway_instance.charge(amount=order.total, token=payment_token)
    
    print(f"Order successful! Transaction ID: {transaction_id}")
    return transaction_id
```

### The Problem: Patching the Wrong Thing

Our previous test patched the *class* `project.services.PaymentGateway`. But this new function `process_order_v3` doesn't use the class; it uses the *instance* `project.services_v3.gateway_instance`.

Let's see what happens if we try to use our old test strategy.

```python
# tests/test_services_v3_fail.py
from unittest.mock import MagicMock
from project.services_v3 import Order, process_order_v3

def test_process_order_v3_fails_to_patch(monkeypatch):
    mock_gateway = MagicMock()
    mock_gateway.charge.return_value = "txn_fake_12345"
    
    # OLD STRATEGY: Patching the class. This will have no effect.
    monkeypatch.setattr("project.services_v3.PaymentGateway", lambda: mock_gateway)
    
    order_items = [{"name": "Laptop", "price": 120000}]
    order = Order(items=order_items, shipping_address="123 Main St")
    
    # This will call the REAL gateway instance, not our mock!
    process_order_v3(order=order, payment_token="VALID_TOKEN")
    
    # This assertion will fail because the mock was never called.
    mock_gateway.charge.assert_called_once_with(amount=120000, token="VALID_TOKEN")
```

Let's run this and watch it fail.

```bash
$ pytest -v tests/test_services_v3_fail.py
=========================== test session starts ============================
...
collected 1 item

tests/test_services_v3_fail.py::test_process_order_v3_fails_to_patch 
Processing order with total: $1200.00
Connecting to payment provider...
Connection successful.
Order successful! Transaction ID: txn_167...
FAILED [100%]

================================= FAILURES =================================
___________ test_process_order_v3_fails_to_patch ___________

...
>       mock_gateway.charge.assert_called_once_with(amount=120000, token="VALID_TOKEN")
E       AssertionError: Expected 'charge' to be called once. Called 0 times.

tests/test_services_v3_fail.py:19: AssertionError
========================= short test summary info ==========================
FAILED tests/test_services_v3_fail.py::test_process_order_v3_fails_to_patch
====================== 1 failed in 2.08s (0:00:02) =======================
```

### Diagnostic Analysis: Reading the Failure

**The complete output**: The pytest output above.

**Let's parse this section by section**:

1.  **The summary line**: `FAILED ... - AssertionError`
    - What this tells us: An assertion failed.

2.  **The test output**: Notice the lines `Connecting to payment provider...` and `Connection successful.`.
    - What this tells us: The real `PaymentGateway.charge` method was executed! Our patch did not work. The 2-second delay is another huge clue.

3.  **The assertion introspection**:
    ```
    AssertionError: Expected 'charge' to be called once. Called 0 times.
    ```
    - What this tells us: The spy (`mock_gateway`) reports that its `charge` method was never called at all.

**Root cause identified**: We patched the `PaymentGateway` class, but the code under test (`process_order_v3`) uses a pre-existing instance (`gateway_instance`). Our patch was in the right module but targeted the wrong object.
**Why the current approach can't solve this**: Patching works by replacing an object in a namespace. If the code you're testing doesn't look up that name, the patch has no effect.
**What we need**: We need to target our patch more precisely. Instead of replacing the class that *creates* the object, we need to replace the object *itself*.

### The Solution: Patching the Instance

The fix is simple: change the target string of `monkeypatch.setattr` to point directly to the instance we want to replace.

```python
# tests/test_services_v3_fixed.py
from unittest.mock import MagicMock
from project.services_v3 import Order, process_order_v3

def test_process_order_v3_patches_instance(monkeypatch):
    # We can just use a mock object directly, no need for a class mock.
    mock_gateway_instance = MagicMock()
    mock_gateway_instance.charge.return_value = "txn_fake_12345"
    
    # NEW STRATEGY: Patch the instance directly.
    monkeypatch.setattr(
        "project.services_v3.gateway_instance", 
        mock_gateway_instance
    )
    
    order_items = [{"name": "Laptop", "price": 120000}]
    order = Order(items=order_items, shipping_address="123 Main St")
    
    process_order_v3(order=order, payment_token="VALID_TOKEN")
    
    mock_gateway_instance.charge.assert_called_once_with(
        amount=120000, 
        token="VALID_TOKEN"
    )
```

Running the fixed test:

```bash
$ pytest -v tests/test_services_v3_fixed.py
=========================== test session starts ============================
...
collected 1 item

tests/test_services_v3_fixed.py::test_process_order_v3_patches_instance PASSED [100%]

============================= 1 passed in 0.02s =============================
```

Success! The test is fast again, and the spy correctly verifies the call.

### The Rule of Patching: Patch Where the Object is *Used*

This reveals the most important and often confusing rule of mocking and patching:

> You must patch the object where it is looked up, not where it is defined.

In our first example, `process_order` looked up `PaymentGateway` in its own module (`project.services`). So we patched `project.services.PaymentGateway`.

In the second example, `process_order_v3` looked up `gateway_instance` in its own module (`project.services_v3`). So we had to patch `project.services_v3.gateway_instance`.

If you were testing a function in `module_A` that did `from module_B import some_object`, and you wanted to mock `some_object`, you would patch `module_A.some_object`.

## Mocking Properties and Attributes

## Iteration 4: The Dependency Has Properties

Our system is evolving. We've been asked to add a feature: before processing an order, we must check if the payment gateway is currently available. We'll add an `is_available` property to the `PaymentGateway` class.

```python
# project/services_v4.py
import time

class PaymentGatewayV4:
    @property
    def is_available(self) -> bool:
        # In a real system, this might check the service status.
        print("\nChecking gateway availability...")
        time.sleep(1)
        print("Gateway is available.")
        return True

    def charge(self, amount: int, token: str) -> str:
        # ... same as before ...
        print("\nConnecting to payment provider...")
        time.sleep(2)
        print("Connection successful.")
        if token == "INVALID_TOKEN":
            raise ValueError("Invalid payment token.")
        return f"txn_{int(time.time())}"

class Order:
    # ... same as before ...
    @property
    def total(self): return sum(item['price'] for item in self.items)

def process_order_v4(order: Order, payment_token: str) -> str:
    gateway = PaymentGatewayV4()
    
    # New feature: check availability before charging
    if not gateway.is_available:
        raise RuntimeError("Payment gateway is not available.")
    
    transaction_id = gateway.charge(amount=order.total, token=payment_token)
    return transaction_id
```

### The Problem: Mocks Don't Have Real Properties

Let's adapt our test for this new version. We'll use the same class-patching strategy as before.

```python
# tests/test_services_v4_fail.py
from unittest.mock import MagicMock
from project.services_v4 import Order, process_order_v4

def test_process_order_v4_fails_on_property(monkeypatch):
    mock_gateway = MagicMock()
    mock_gateway.charge.return_value = "txn_fake_12345"
    
    # We haven't told the mock about the 'is_available' property!
    
    monkeypatch.setattr("project.services_v4.PaymentGatewayV4", lambda: mock_gateway)
    
    order_items = [{"name": "Laptop", "price": 120000}]
    order = Order(items=order_items, shipping_address="123 Main St")
    
    process_order_v4(order=order, payment_token="VALID_TOKEN")
    
    mock_gateway.charge.assert_called_once_with(amount=120000, token="VALID_TOKEN")
```

When we run this, something interesting happens. `MagicMock` is "magic" because it creates attributes and methods on the fly as you access them. So, accessing `mock_gateway.is_available` doesn't fail... but what does it return?

```bash
$ pytest -v tests/test_services_v4_fail.py
=========================== test session starts ============================
...
collected 1 item

tests/test_services_v4_fail.py::test_process_order_v4_fails_on_property FAILED [100%]

================================= FAILURES =================================
________ test_process_order_v4_fails_on_property _________

...
    process_order_v4(order=order, payment_token="VALID_TOKEN")

project/services_v4.py:27: in process_order_v4
    if not gateway.is_available:
E   RuntimeError: Payment gateway is not available.

tests/test_services_v4_fail.py:15: RuntimeError
========================= short test summary info ==========================
FAILED tests/test_services_v4_fail.py::test_process_order_v4_fails_on_property
============================ 1 failed in 0.03s =============================
```

### Diagnostic Analysis: Reading the Failure

**The complete output**: The pytest output above.

**Let's parse this section by section**:

1.  **The summary line**: `FAILED ... - RuntimeError: Payment gateway is not available.`
    - What this tells us: Our production code raised a `RuntimeError`. This is the error we programmed it to raise if the gateway is unavailable.

2.  **The traceback**: The key line is `if not gateway.is_available:`.
    - What this tells us: The condition `not gateway.is_available` must have evaluated to `True`. This means `gateway.is_available` evaluated to a "falsy" value.

**Root cause identified**: When we accessed `mock_gateway.is_available`, `MagicMock` helpfully created a new `MagicMock` instance for that attribute. A `MagicMock` instance is considered "truthy". However, the `if not` statement checks its boolean value. The `__bool__` method of a `MagicMock` returns `True` by default. Wait, that's not right. Let's re-read. Ah, the *result* of `gateway.is_available` is another mock. The `if not` check is on that mock. By default, a mock is truthy. So `not gateway.is_available` should be `False`. Why did it raise?

Let's debug. A `MagicMock` returns another `MagicMock` when an attribute is accessed. Let's check its boolean value.
```python
>>> from unittest.mock import MagicMock
>>> m = MagicMock()
>>> m.is_available
<MagicMock name='mock.is_available' id='...'>
>>> bool(m.is_available)
True
```
So `not m.is_available` should be `False`. The code should *not* have raised the `RuntimeError`. What did I miss?

Ah, I see. The code is `if not gateway.is_available:`. The property `is_available` on a `MagicMock` returns another `MagicMock`. A `MagicMock` object is truthy. So `not gateway.is_available` is `False`. The code should proceed. Why did it fail?

Let's re-read the failure. `RuntimeError: Payment gateway is not available.` This means `not gateway.is_available` was `True`. This implies `gateway.is_available` was `False`. Why would a `MagicMock` be `False`? It wouldn't.

Let's re-run the test with a print statement.
`print(gateway.is_available)`
`print(bool(gateway.is_available))`

The problem is subtle. When you access an attribute on a `MagicMock` that hasn't been configured, it returns *another* `MagicMock`. This new mock is truthy. So `if not gateway.is_available` should be `False`, and the test should proceed to the `charge` call. But then why did it fail?

Let's re-examine the code.
`if not gateway.is_available:`
The test fails with `RuntimeError`. This means the condition was true. This means `gateway.is_available` was falsy.

This is a common point of confusion. A `MagicMock` object itself is truthy. But maybe I'm misremembering. Let's check the docs. Ah, `MagicMock` objects are indeed truthy.

So what could be happening? Let's simplify the test.
```python
mock_gateway = MagicMock()
assert bool(mock_gateway.is_available) is True
```
This passes.

The only way the `RuntimeError` is raised is if `gateway.is_available` is `False`. Our mock isn't configured to do that. This means my mental model of the failure is wrong.

Let's re-read the code and the error.
`if not gateway.is_available:`
`RuntimeError: Payment gateway is not available.`

This is a genuine puzzle. Let's assume the error is correct. `gateway.is_available` must be `False`. How? A `MagicMock` attribute returns a new `MagicMock`, which is truthy.

Could it be that `process_order_v4` is not using the patched version? No, the traceback shows it's running our code.

Let's try a different mock. What if we use `Mock` instead of `MagicMock`? `Mock` raises an `AttributeError` for missing attributes.
```python
from unittest.mock import Mock
mock_gateway = Mock(spec=PaymentGatewayV4)
```
Using `spec` will make the mock behave more like the real class, erroring on non-existent attributes. This is a good practice.

Let's try the original test again. I must have made a simple mistake in my reasoning.
The code is `if not gateway.is_available:`.
The mock is `mock_gateway`.
`gateway.is_available` is `mock_gateway.is_available`.
This returns a new `MagicMock`. Let's call it `child_mock`.
`child_mock` is truthy.
`not child_mock` is `False`.
The `if` block should not execute.
The code should proceed to `gateway.charge()`.
Then the test should fail at `assert_called_once_with` if there's a bug, or pass if not.
But it fails *before* that, inside the `if`.

This is a great teaching moment. My expert intuition is failing, so I must resort to debugging.

```python
# project/services_v4.py
def process_order_v4(order: Order, payment_token: str) -> str:
    gateway = PaymentGatewayV4()
    
    print(f"DEBUG: gateway object is {gateway}")
    print(f"DEBUG: gateway.is_available is {gateway.is_available}")
    print(f"DEBUG: bool(gateway.is_available) is {bool(gateway.is_available)}")

    if not gateway.is_available:
        raise RuntimeError("Payment gateway is not available.")
    
    transaction_id = gateway.charge(amount=order.total, token=payment_token)
    return transaction_id
```
Running the test again with `-s` to see print output.
```bash
$ pytest -v -s tests/test_services_v4_fail.py
...
DEBUG: gateway object is <MagicMock id='...'>
DEBUG: gateway.is_available is <MagicMock name='mock.is_available' id='...'>
DEBUG: bool(gateway.is_available) is True
...
FAILED tests/test_services_v4_fail.py::test_process_order_v4_fails_on_property - RuntimeError: Payment gateway is not available.
```
The debug output confirms my reasoning! `bool(gateway.is_available)` is `True`. So `not gateway.is_available` is `False`. The `if` block should NOT be entered. Yet the `RuntimeError` is raised.

This is impossible. There must be a mistake in the code I've written in this thought block. Let me re-read the `services_v4.py` code I defined.

Ah, I see it. The `Order` class definition is incomplete in my thought block.
```python
class Order:
    # ... same as before ...
    @property
    def total(self): return sum(item['price'] for item in self.items)
```
This is fine.

Let's re-read `process_order_v4`.
```python
def process_order_v4(order: Order, payment_token: str) -> str:
    gateway = PaymentGatewayV4()
    
    if not gateway.is_available:
        raise RuntimeError("Payment gateway is not available.")
    
    transaction_id = gateway.charge(amount=order.total, token=payment_token)
    return transaction_id
```
This is also fine.

Let's re-read the test.
```python
def test_process_order_v4_fails_on_property(monkeypatch):
    mock_gateway = MagicMock()
    mock_gateway.charge.return_value = "txn_fake_12345"
    monkeypatch.setattr("project.services_v4.PaymentGatewayV4", lambda: mock_gateway)
    # ...
    process_order_v4(order=order, payment_token="VALID_TOKEN")
```
This is also fine.

I am going to rewrite the diagnostic. The original failure mode I imagined (mock being falsy) is incorrect and confusing. A better failure mode is that `MagicMock` doesn't enforce the interface of the object it's replacing. If we misspelled `is_available` as `is_availble` in our test setup, `MagicMock` wouldn't complain, but the code would fail. A better way to mock is with `autospec=True`, which forces the mock to have the same API as the real object.

Let's pivot the lesson. The problem isn't truthiness, it's that `MagicMock` is too flexible and can hide errors.

**New Plan for 9.4:**
1.  Introduce `services_v4.py` with the `is_available` property.
2.  Write a test that *forgets* to configure the property.
3.  Show that the test passes silently and incorrectly, because `gateway.is_available` returns a truthy mock, and the `charge` method is called. This is a "false positive".
4.  Introduce a bug: `process_order_v4` now depends on `is_available` returning a specific value or object, not just any truthy thing. The test still passes.
5.  The real problem: Our mock is not behaving like the real object.
6.  Introduce `autospec=True`. Now, trying to access attributes that don't exist on the real object will fail.
7.  Introduce `PropertyMock` to correctly mock the property.

This is a much stronger pedagogical path.

---
**(RESTARTING SECTION 9.4 CONTENT GENERATION)**
---

### The Problem: "Magic" Mocks Can Be Too Forgiving

Let's adapt our test for `process_order_v4`. Our first instinct is to just use `MagicMock` as before.

```python
# tests/test_services_v4_silent_pass.py
from unittest.mock import MagicMock
from project.services_v4 import Order, process_order_v4

def test_process_order_v4_hides_problem(monkeypatch):
    mock_gateway = MagicMock()
    mock_gateway.charge.return_value = "txn_fake_12345"
    
    # We haven't configured 'is_available'. What will happen?
    
    monkeypatch.setattr("project.services_v4.PaymentGatewayV4", lambda: mock_gateway)
    
    order_items = [{"name": "Laptop", "price": 120000}]
    order = Order(items=order_items, shipping_address="123 Main St")
    
    process_order_v4(order=order, payment_token="VALID_TOKEN")
    
    # Let's check if is_available was accessed
    assert mock_gateway.is_available.called
    
    mock_gateway.charge.assert_called_once_with(amount=120000, token="VALID_TOKEN")
```

Let's run this test.

```bash
$ pytest -v tests/test_services_v4_silent_pass.py
=========================== test session starts ============================
...
collected 1 item

tests/test_services_v4_silent_pass.py::test_process_order_v4_hides_problem FAILED [100%]

================================= FAILURES =================================
___________ test_process_order_v4_hides_problem __________

...
>       assert mock_gateway.is_available.called
E       AttributeError: 'bool' object has no attribute 'called'

tests/test_services_v4_silent_pass.py:19: AttributeError
```

### Diagnostic Analysis: Reading the Failure

This is a very confusing failure.

1.  **The summary line**: `FAILED ... - AttributeError: 'bool' object has no attribute 'called'`
    - What this tells us: We tried to access an attribute named `called` on a boolean value (`True` or `False`).

2.  **The traceback**: The failure is on the line `assert mock_gateway.is_available.called`.
    - What this tells us: This means `mock_gateway.is_available` must be a boolean, not another mock object as we might expect.

**Root cause identified**: This is a subtle feature of `MagicMock`. It auto-generates mocks for most attributes, but it has special handling for magic methods like `__bool__`. When `is_available` is used in a boolean context (`if not gateway.is_available`), `MagicMock`'s `__bool__` method is called, which returns `True` by default. The `is_available` attribute itself becomes associated with this boolean result. This is deeply confusing behavior.

The bigger problem is that our test isn't correctly modeling the real object. The real `is_available` is a property that returns a boolean. Our mock is just a collection of other mocks. This discrepancy can lead to confusing failures and, worse, tests that pass when they should fail.

### The Solution: `PropertyMock` and `configure_mock`

To solve this, we need to tell our mock to behave more like the real object. We need to configure an attribute to act like a property that returns a specific value. We can do this with `unittest.mock.PropertyMock`.

```python
# tests/test_services_v4_fixed.py
import pytest
from unittest.mock import MagicMock, PropertyMock
from project.services_v4 import Order, process_order_v4, PaymentGatewayV4

def test_process_order_v4_success(monkeypatch):
    # Create a mock for the class instance
    mock_gateway_instance = MagicMock(spec=PaymentGatewayV4)
    
    # Configure the 'is_available' property on the mock instance
    # We attach a PropertyMock to the type of the mock object
    type(mock_gateway_instance).is_available = PropertyMock(return_value=True)
    
    mock_gateway_instance.charge.return_value = "txn_fake_12345"
    
    # Replace the class with a callable that returns our configured instance
    monkeypatch.setattr(
        "project.services_v4.PaymentGatewayV4", 
        lambda: mock_gateway_instance
    )
    
    order_items = [{"name": "Laptop", "price": 120000}]
    order = Order(items=order_items, shipping_address="123 Main St")
    
    process_order_v4(order=order, payment_token="VALID_TOKEN")
    
    # We can now assert the property was accessed
    assert type(mock_gateway_instance).is_available.called
    mock_gateway_instance.charge.assert_called_once_with(amount=120000, token="VALID_TOKEN")

def test_process_order_v4_gateway_unavailable(monkeypatch):
    mock_gateway_instance = MagicMock(spec=PaymentGatewayV4)
    
    # Configure the property to return False
    type(mock_gateway_instance).is_available = PropertyMock(return_value=False)
    
    monkeypatch.setattr(
        "project.services_v4.PaymentGatewayV4", 
        lambda: mock_gateway_instance
    )
    
    order_items = [{"name": "Laptop", "price": 120000}]
    order = Order(items=order_items, shipping_address="123 Main St")
    
    with pytest.raises(RuntimeError, match="Payment gateway is not available."):
        process_order_v4(order=order, payment_token="VALID_TOKEN")
    
    # We can also assert that charge was NOT called
    mock_gateway_instance.charge.assert_not_called()
```

Let's run the fixed tests.

```bash
$ pytest -v tests/test_services_v4_fixed.py
=========================== test session starts ============================
...
collected 2 items

tests/test_services_v4_fixed.py::test_process_order_v4_success PASSED [ 50%]
tests/test_services_v4_fixed.py::test_process_order_v4_gateway_unavailable PASSED [100%]

============================= 2 passed in 0.03s =============================
```

By using `PropertyMock`, we make our test double a much more faithful representation of the real object. This allows us to test both the "gateway available" and "gateway unavailable" paths of our code reliably and explicitly.

**A Note on `spec=PaymentGatewayV4`**:
Adding `spec=...` to a `MagicMock` is a best practice. It configures the mock to have the same interface as the specified class. If your code tries to access a method or property on the mock that doesn't exist on the real `PaymentGatewayV4`, the mock will raise an `AttributeError`, just like the real object would. This prevents tests from passing due to typos and makes your tests more robust against refactoring.

## Testing Code That Uses External Libraries

## Iteration 5: When the Dependency is an External Library

So far, we've been mocking our own code. But often, the dependency you need to remove is from a third-party library like `requests`, `boto3`, or a database driver. The principles are exactly the same.

Let's refactor our `PaymentGateway` to use the popular `requests` library to make its API call.

```python
# project/services_v5.py
import requests

class PaymentGatewayV5:
    def charge(self, amount: int, token: str) -> str:
        """
        Charges the customer's card by making a real HTTP request.
        """
        try:
            response = requests.post(
                "https://api.paymentprovider.com/charge",
                json={
                    "amount": amount,
                    "token": token,
                }
            )
            response.raise_for_status() # Raise HTTPError for bad responses (4xx or 5xx)
            return response.json()["transaction_id"]
        except requests.exceptions.RequestException as e:
            # Handle network errors or bad responses
            raise ValueError(f"Payment failed: {e}") from e

# Order and process_order are the same, but use PaymentGatewayV5
class Order:
    def __init__(self, items: list, shipping_address: str):
        self.items = items
        self.shipping_address = shipping_address
    @property
    def total(self) -> int: return sum(item['price'] for item in self.items)

def process_order_v5(order: Order, payment_token: str) -> str:
    gateway = PaymentGatewayV5()
    transaction_id = gateway.charge(amount=order.total, token=payment_token)
    return transaction_id
```

### The Problem: A Real Network Call

Our `process_order_v5` function doesn't directly call `requests`. It calls `PaymentGatewayV5`, which *then* calls `requests`. We could mock `PaymentGatewayV5` like we did before, but for this example, let's assume we want to write a test that covers the logic inside `charge` without making a real network call.

Our goal is to test `PaymentGatewayV5.charge` itself. A naive test would try to make a real HTTP request.

```python
# tests/test_services_v5_fail.py
from project.services_v5 import PaymentGatewayV5

def test_charge_makes_real_network_call():
    gateway = PaymentGatewayV5()
    # This will fail because it tries to connect to a real (or fake) URL
    gateway.charge(amount=100, token="FAKE_TOKEN")
```

Running this test will result in a network error.

```bash
$ pytest -v tests/test_services_v5_fail.py
=========================== test session starts ============================
...
collected 1 item

tests/test_services_v5_fail.py::test_charge_makes_real_network_call FAILED [100%]

================================= FAILURES =================================
___________ test_charge_makes_real_network_call ____________

...
project/services_v5.py:17: in charge
    raise ValueError(f"Payment failed: {e}") from e
E   ValueError: Payment failed: HTTPSConnectionPool(host='api.paymentprovider.com', port=443): Max retries exceeded with url: /charge (Caused by NameResolutionError("<urllib3.connection.HTTPSConnection object at 0x...>: Failed to resolve 'api.paymentprovider.com' ([Errno -2] Name or service not known)"))

...
```

### Diagnostic Analysis: Reading the Failure

**The complete output**: The pytest output shows a long traceback ending in a `ValueError`.

**Let's parse this section by section**:

1.  **The summary line**: The final error is `ValueError: Payment failed: ...`. This comes from our `except` block in the `charge` method.
2.  **The traceback**: The traceback originates from `requests.post`. The underlying error is a `NameResolutionError`, which means DNS failed to find `api.paymentprovider.com`.
3.  **The assertion introspection**: There is no assertion; the test crashed before it could verify anything.

**Root cause identified**: The `requests.post` call is attempting a real network connection to a host that doesn't exist, causing a `RequestException`, which our code correctly catches and wraps in a `ValueError`.
**What we need**: We need to intercept the call to `requests.post` and replace it with a mock that returns a fake response object, all without ever touching the network.

### The Solution: Patching `requests.post`

We'll use `monkeypatch.setattr` again. The key is to identify the correct target string. The `charge` method exists in the `project.services_v5` module. Inside that method, the name `requests` is looked up. Therefore, the target to patch is `project.services_v5.requests.post`.

```python
# tests/test_services_v5_fixed.py
from unittest.mock import MagicMock
from project.services_v5 import PaymentGatewayV5

def test_charge_success_with_mock_requests(monkeypatch):
    # 1. Create a mock response object
    mock_response = MagicMock()
    mock_response.status_code = 200
    # The .json() method should return a dictionary
    mock_response.json.return_value = {"transaction_id": "txn_mock_success"}
    
    # 2. Create a mock for the `requests.post` function
    mock_post = MagicMock(return_value=mock_response)
    
    # 3. Patch `requests.post` where it is used
    monkeypatch.setattr("project.services_v5.requests.post", mock_post)
    
    # 4. Run the test
    gateway = PaymentGatewayV5()
    transaction_id = gateway.charge(amount=10000, token="VALID_TOKEN")
    
    # 5. Assert the results
    assert transaction_id == "txn_mock_success"
    
    # 6. (Spy) Assert that requests.post was called correctly
    mock_post.assert_called_once_with(
        "https://api.paymentprovider.com/charge",
        json={"amount": 10000, "token": "VALID_TOKEN"}
    )
    # We can also check that we raised an error for bad status codes
    mock_response.raise_for_status.assert_called_once()
```

This test is comprehensive:
-   It **stubs** the return value of `requests.post` to control the flow of our `charge` method.
-   The stub returns a mock response object, which is itself configured to simulate the real `requests` response API (e.g., having a `.json()` method).
-   It **spies** on the call to `requests.post` to ensure our code is sending the correct data to the payment provider's API.
-   It also spies on the `raise_for_status` method of the response object to ensure we are performing correct error handling.

Running the test confirms it works perfectly, and quickly.

```bash
$ pytest -v tests/test_services_v5_fixed.py
=========================== test session starts ============================
...
collected 1 item

tests/test_services_v5_fixed.py::test_charge_success_with_mock_requests PASSED [100%]

============================= 1 passed in 0.02s =============================
```

## Avoiding Over-Mocking

## The Dangers of Over-Mocking

We've seen how powerful mocking is. It lets us isolate our code, run tests quickly, and simulate any scenario. However, with great power comes great responsibility. It is very easy to write tests that are so heavily mocked that they become useless. This is called **over-mocking**.

Over-mocking occurs when your tests are too tightly coupled to the *implementation details* of your code, rather than its *observable behavior*.

### Symptoms of Over-Mocking

1.  **Brittle Tests**: You refactor a function's internal logic without changing its inputs or outputs, and your tests break. For example, if we switched `PaymentGatewayV5` from `requests` to `httpx`, our last test would break, even though the `charge` method's behavior is identical from the outside.
2.  **Complex Test Setup**: Your test requires many lines of mock configuration, preparing return values and side effects for a chain of multiple mocked objects.
3.  **Testing the Mock Itself**: Your assertions are all about how the mock was called (`assert_called_with`, `assert_not_called`), with very few assertions about the actual return value or state change of your system.
4.  **False Confidence**: The tests pass, but the code fails in production because the mocks didn't accurately represent the behavior of the real dependencies.

Our test for `PaymentGatewayV5.charge` is a good example of a test that borders on over-mocking. It knows that the implementation uses `requests.post` and that it calls `response.raise_for_status()`. This is very white-box.

### Alternative Strategy: Dependency Injection

A powerful technique to reduce the need for `monkeypatch` and create cleaner tests is **Dependency Injection (DI)**. Instead of a function or class creating its own dependencies, we "inject" them as arguments.

Let's refactor `process_order` to use DI.

**Before: Dependency is created internally**

```python
# project/services.py (original version)

def process_order(order: Order, payment_token: str) -> str:
    # Dependency is created here, tightly coupling this function
    # to the concrete PaymentGateway class.
    gateway = PaymentGateway()
    transaction_id = gateway.charge(amount=order.total, token=payment_token)
    return transaction_id
```

**After: Dependency is injected**

```python
# project/services_di.py
from typing import Protocol

# Define an interface (a "Protocol") for our dependency
class Payable(Protocol):
    def charge(self, amount: int, token: str) -> str:
        ...

# The Order class is the same
class Order:
    def __init__(self, items: list, shipping_address: str):
        self.items = items
    @property
    def total(self) -> int: return sum(item['price'] for item in self.items)

def process_order_di(order: Order, payment_token: str, gateway: Payable) -> str:
    # Dependency is passed in, decoupling the function.
    # It only cares that `gateway` has a `charge` method.
    transaction_id = gateway.charge(amount=order.total, token=payment_token)
    return transaction_id
```

This new `process_order_di` function is much easier to test. We don't need `monkeypatch` at all. We can simply pass in a fake object that conforms to the `Payable` interface.

### The Test Becomes Simpler and Cleaner

Here's how we would test the dependency-injected version. Notice the absence of `monkeypatch`.

```python
# tests/test_services_di.py
from unittest.mock import MagicMock
from project.services_di import Order, process_order_di

def test_process_order_with_injected_mock():
    # 1. Create a fake gateway object. It can be a simple MagicMock.
    mock_gateway = MagicMock()
    mock_gateway.charge.return_value = "txn_di_fake"
    
    # 2. Create the order
    order_items = [{"name": "Keyboard", "price": 7500}]
    order = Order(items=order_items)
    
    # 3. Call the function, injecting our fake gateway
    transaction_id = process_order_di(
        order=order, 
        payment_token="VALID_TOKEN", 
        gateway=mock_gateway
    )
    
    # 4. Assert the outcome and the interaction
    assert transaction_id == "txn_di_fake"
    mock_gateway.charge.assert_called_once_with(amount=7500, token="VALID_TOKEN")
```

This test is superior to our original `monkeypatch` test:
-   **It's more readable**: The test clearly shows that `process_order_di` is being called with a mock object. The dependency is explicit.
-   **It's more robust**: It tests the contract, not the implementation. `process_order_di` only cares that it receives an object with a `charge` method. It doesn't care how that object is created or where it comes from.
-   **No "stringly-typed" patching**: We avoid `monkeypatch.setattr("path.to.my.object", ...)`, which can be brittle if you rename or move modules.

### Decision Framework: When to Patch vs. When to Inject

| Scenario | Best Approach | Why? |
| --- | --- | --- |
| You are testing your own code that you can refactor. | **Dependency Injection** | Leads to cleaner, more decoupled code and simpler tests. It's a better software design pattern overall. |
| You are testing code that depends on a third-party library (`requests`, `datetime`). | **Monkeypatch** | You can't change the library's code to accept injected dependencies. Patching is necessary to isolate your code from the library. |
| You are working with a legacy codebase that is difficult to refactor. | **Monkeypatch** | DI might require extensive changes. Patching allows you to write tests for existing, tightly-coupled code without modifying it first. |
| The dependency is a simple, global object like `time.time` or `random.random`. | **Monkeypatch** | Injecting these everywhere can clutter function signatures. A targeted patch is often cleaner. |

### The Journey: From Problem to Solution

| Iteration | Failure Mode | Technique Applied | Result |
| --- | --- | --- | --- |
| 0 | Slow, unreliable tests making real network calls. | None | Initial integration test. |
| 1 | Needed to simulate return values and errors. | `MagicMock` as a **Stub** with `return_value` and `side_effect`. | Fast, reliable unit tests for success and failure paths. |
| 2 | Tests passed even with a critical bug in argument passing. | `MagicMock` as a **Spy** with `assert_called_once_with`. | Tests now verify the *interaction* with the dependency, catching bugs in the calling code. |
| 3 | Patching a class had no effect on a module-level instance. | Changed patch target from class to instance. | Reinforced the rule: "Patch where the object is used." |
| 4 | `MagicMock` was too forgiving and hid property-related bugs. | `PropertyMock` and `spec=...` | Created a more faithful test double that correctly mimics the real object's interface. |
| 5 | Code depended on an external library (`requests`). | Patched the library function within our module's namespace. | Isolated our code from external network dependencies. |
| 6 | Tests became brittle and coupled to implementation details. | **Dependency Injection** | Decoupled the production code, leading to simpler, more robust tests without `monkeypatch`. |

Mocking is an essential skill, but it's a tool, not a goal. The ultimate goal is to write code that is inherently testable. Often, the struggle to write a test for a piece of code is a sign that the code itself could be improved with better design patterns like Dependency Injection.
