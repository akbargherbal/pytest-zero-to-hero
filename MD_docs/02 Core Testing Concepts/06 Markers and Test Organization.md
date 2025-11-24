# Chapter 6: Markers and Test Organization

## What Are Markers?

## What Are Markers?

As a test suite grows, you'll inevitably face an organization problem. You'll have fast unit tests, slower integration tests, and perhaps even end-to-end tests that take minutes to run. During development, you might want to run only the fast unit tests. Before a release, you might want to run everything *except* tests for features that are currently disabled.

How can you manage this complexity?

Pytest's solution is **markers**. A marker is a metadata tag you can apply to a test function or class. It doesn't change what the test does, but it gives you powerful ways to group, filter, and manage your tests from the command line.

Think of markers like labels in your email inbox (`work`, `urgent`, `personal`) or tags on a blog post (`python`, `testing`, `pytest`). The content remains the same, but the tags allow you to find and act on specific items easily.

### Phase 1: Establish the Reference Implementation

Let's build our anchor example for this chapter. We'll model a simple e-commerce order processing system. The system has different components, and we'll write different kinds of tests for them.

First, the application code we'll be testing.

**The Anchor Example: `commerce_system.py`**

```python
# commerce_system.py

import time
import warnings

class Product:
    def __init__(self, name: str, price: float):
        self.name = name
        self.price = price

class Order:
    def __init__(self, products: list[Product]):
        if not products:
            raise ValueError("Cannot create an order with no products.")
        self.products = products
        self._discount_applied = False

    def total(self) -> float:
        """Calculates the total price of products in the order."""
        base_total = sum(p.price for p in self.products)
        return base_total

    def apply_discount(self, percentage: int):
        """Applies a percentage discount to the order total."""
        # Known bug: This logic is incorrect, it should modify the total.
        # We will use this to demonstrate xfail later.
        if not 0 < percentage <= 100:
            raise ValueError("Discount must be between 1 and 100.")
        if self._discount_applied:
            raise RuntimeError("Discount has already been applied.")
        
        # Buggy implementation for demonstration purposes
        self.total() * ((100 - percentage) / 100)
        self._discount_applied = True

def connect_to_db():
    """Simulates a slow database connection."""
    print("\n(Connecting to database...)")
    time.sleep(1)
    print("(Connection successful.)")
    return {"status": "connected"}

def process_payment_via_api(total: float):
    """Simulates a very slow external API call."""
    warnings.warn("This payment gateway is deprecated.", DeprecationWarning)
    print(f"\n(Calling external payment API for ${total:.2f}...)")
    time.sleep(2)
    print("(API call successful.)")
    return {"status": "paid", "amount": total}
```

Now, let's write a test file with a mix of tests for this system.

**The Initial Test Suite: `test_commerce_initial.py`**

```python
# test_commerce_initial.py

from commerce_system import Order, Product, connect_to_db, process_payment_via_api

# A fast, simple unit test
def test_order_total():
    """Tests the total calculation for a simple order."""
    products = [Product("Laptop", 1200.00), Product("Mouse", 25.00)]
    order = Order(products)
    assert order.total() == 1225.00

# A test for a known bug
def test_order_discount_logic():
    """Tests the discount application logic, which is known to be buggy."""
    products = [Product("Keyboard", 100.00), Product("Mouse", 25.00)]
    order = Order(products)
    order.apply_discount(10)
    # This will fail because the discount logic is flawed
    assert order.total() == 112.50

# A slower integration test
def test_database_connection():
    """Tests the function that connects to the database."""
    conn = connect_to_db()
    assert conn["status"] == "connected"

# A very slow API test
def test_payment_api_call():
    """Tests the external payment API call."""
    result = process_payment_via_api(total=150.00)
    assert result["status"] == "paid"
    assert result["amount"] == 150.00
```

### The Problem: An Unorganized Suite

Let's run this suite.

```bash
$ pytest -v test_commerce_initial.py
=========================== test session starts ============================
...
collected 4 items

test_commerce_initial.py::test_order_total PASSED                     [ 25%]
test_commerce_initial.py::test_order_discount_logic FAILED            [ 50%]
test_commerce_initial.py::test_database_connection 
(Connecting to database...)
(Connection successful.)
PASSED [ 75%]
test_commerce_initial.py::test_payment_api_call 
(Calling external payment API for $150.00...)
.../test_commerce_initial.py:31: DeprecationWarning: This payment gateway is deprecated.
  result = process_payment_via_api(total=150.00)
(API call successful.)
PASSED [100%]

================================= FAILURES =================================
________________________ test_order_discount_logic _________________________

    def test_order_discount_logic():
        """Tests the discount application logic, which is known to be buggy."""
        products = [Product("Keyboard", 100.00), Product("Mouse", 25.00)]
        order = Order(products)
        order.apply_discount(10)
        # This will fail because the discount logic is flawed
>       assert order.total() == 112.50
E       assert 125.0 == 112.5
E        +  where 125.0 = <bound method Order.total of <commerce_system.Order object at 0x...>>()
E        +    where <bound method Order.total of <commerce_system.Order object at 0x...>> = <commerce_system.Order object at 0x...>.total

test_commerce_initial.py:18: AssertionError
========================= short test summary info ==========================
FAILED test_commerce_initial.py::test_order_discount_logic - assert 125.0 == 112.5
================== 1 failed, 3 passed in 3.05s ===================
```

We have several problems here:

1.  **One Failure Halts the Build:** The `test_order_discount_logic` is failing due to a known bug. This marks the entire test run as a failure, which might block a CI/CD pipeline.
2.  **Everything Runs:** The entire suite took over 3 seconds to run. The database and API tests are slow. During rapid development, we don't want to wait for these every single time.
3.  **Noisy Warnings:** The `DeprecationWarning` clutters the output. In a large suite, this noise can hide important information.

We have no way to tell pytest, "Run the fast tests," or "Ignore the known failure for now." This is the problem that markers are designed to solve.

## Built-in Markers (skip, xfail, filterwarnings)

## Built-in Markers

Pytest provides several useful markers out of the box to handle common testing scenarios like known bugs, temporarily disabled tests, and unwanted warnings. Let's apply them to our commerce test suite to solve the problems we just identified.

### Iteration 1: Handling a Known Bug with `@pytest.mark.xfail`

Our current test suite fails because of a known bug in the `apply_discount` method. We don't want to delete the test—it's valuable documentation of the bug—but we also don't want it to fail our build.

**Current Limitation:** A legitimate test is failing due to a bug in the application code, causing the entire test suite to be marked as `FAILED`.

**New Scenario:** We need to inform pytest that we *expect* this test to fail.

**Failure Demonstration:** As we saw above, running `pytest` results in a clear `FAILED` status.

### Diagnostic Analysis: Reading the Failure

**The complete output**:
```bash
================================= FAILURES =================================
________________________ test_order_discount_logic _________________________

    def test_order_discount_logic():
        """Tests the discount application logic, which is known to be buggy."""
        products = [Product("Keyboard", 100.00), Product("Mouse", 25.00)]
        order = Order(products)
        order.apply_discount(10)
        # This will fail because the discount logic is flawed
>       assert order.total() == 112.50
E       assert 125.0 == 112.5

test_commerce_initial.py:18: AssertionError
========================= short test summary info ==========================
FAILED test_commerce_initial.py::test_order_discount_logic - assert 125.0 == 112.5
```

**Let's parse this section by section**:

1.  **The summary line**: `FAILED test_commerce_initial.py::test_order_discount_logic - assert 125.0 == 112.5`
    -   What this tells us: The test failed because of a direct `AssertionError`. The values did not match.

2.  **The traceback**: The traceback points directly to the line `assert order.total() == 112.50`.
    -   What this tells us: The failure is in the test's final check.

3.  **The assertion introspection**: `E assert 125.0 == 112.5`
    -   What this tells us: Pytest shows us the exact values being compared. The `order.total()` method returned `125.0`, but we expected `112.5`.

**Root cause identified**: The `apply_discount` method does not actually change the value returned by `order.total()`.
**Why the current approach can't solve this**: We can't fix the test without fixing the application code. But if the bug fix is scheduled for a later time, the test will continue to fail the build.
**What we need**: A way to tell pytest: "Run this test, but I expect it to fail. Don't count this failure against me."

### Technique Introduced: `@pytest.mark.xfail`

The `@pytest.mark.xfail` (expected failure) marker does exactly this. It tells pytest to run the test, but if it fails in a way we expect (like an `AssertionError`), the test run as a whole will still pass.

Let's apply it. We'll rename our file to `test_commerce_marked.py` to track our progress.

### Solution Implementation

**Before (`test_commerce_initial.py`):**
```python
def test_order_discount_logic():
    """Tests the discount application logic, which is known to be buggy."""
    products = [Product("Keyboard", 100.00), Product("Mouse", 25.00)]
    order = Order(products)
    order.apply_discount(10)
    # This will fail because the discount logic is flawed
    assert order.total() == 112.50
```

**After (`test_commerce_marked.py`):**
```python
import pytest
from commerce_system import Order, Product, connect_to_db, process_payment_via_api

# ... other tests ...

@pytest.mark.xfail(reason="Discount logic is buggy, see TICKET-123")
def test_order_discount_logic():
    """Tests the discount application logic, which is known to be buggy."""
    products = [Product("Keyboard", 100.00), Product("Mouse", 25.00)]
    order = Order(products)
    order.apply_discount(10)
    # This will fail because the discount logic is flawed
    assert order.total() == 112.50

# ... other tests ...
```
Note that we added a `reason` argument. This is excellent practice for documenting *why* the test is expected to fail.

### Verification

Let's run the new file.

```bash
$ pytest -v test_commerce_marked.py
=========================== test session starts ============================
...
collected 4 items

test_commerce_marked.py::test_order_total PASSED                      [ 25%]
test_commerce_marked.py::test_order_discount_logic XFAIL              [ 50%]
test_commerce_marked.py::test_database_connection 
(Connecting to database...)
(Connection successful.)
PASSED [ 75%]
test_commerce_marked.py::test_payment_api_call 
(Calling external payment API for $150.00...)
.../test_commerce_marked.py:32: DeprecationWarning: This payment gateway is deprecated.
  result = process_payment_via_api(total=150.00)
(API call successful.)
PASSED [100%]

==================== 3 passed, 1 xfailed in 3.06s ====================
```

**Expected vs. Actual Improvement:** The test suite now passes! The summary shows `3 passed, 1 xfailed`. The known failure is acknowledged but no longer breaks the build. This is a huge win for CI/CD stability.

A fascinating side effect of `xfail` is that if the test *unexpectedly passes* (e.g., someone fixes the bug), pytest will report it as `XPASS`. This is a signal that your test and code are out of sync and the `xfail` marker should be removed.

### Iteration 2: Temporarily Disabling a Test with `@pytest.mark.skip`

Now, imagine the database team is refactoring the schema, and `connect_to_db()` is temporarily broken. The `test_database_connection` is guaranteed to fail, and running it just wastes time.

**Current Limitation:** We have a test for a feature that is temporarily unavailable. Running it is pointless and slows down the test suite.

**What we need:** A way to tell pytest: "Don't even bother running this test for now."

### Technique Introduced: `@pytest.mark.skip`

The `@pytest.mark.skip` marker tells pytest to not execute a test function at all. Like `xfail`, it's best practice to provide a `reason`.

### Solution Implementation

Let's add it to `test_database_connection`.

**Before:**
```python
# A slower integration test
def test_database_connection():
    """Tests the function that connects to the database."""
    conn = connect_to_db()
    assert conn["status"] == "connected"
```

**After (`test_commerce_marked.py`):**
```python
# ... other tests ...

# A slower integration test
@pytest.mark.skip(reason="Database schema is being refactored.")
def test_database_connection():
    """Tests the function that connects to the database."""
    conn = connect_to_db()
    assert conn["status"] == "connected"

# ... other tests ...
```

### Verification

```bash
$ pytest -v test_commerce_marked.py
=========================== test session starts ============================
...
collected 4 items

test_commerce_marked.py::test_order_total PASSED                      [ 25%]
test_commerce_marked.py::test_order_discount_logic XFAIL              [ 50%]
test_commerce_marked.py::test_database_connection SKIPPED (Database schema is being refactored.) [ 75%]
test_commerce_marked.py::test_payment_api_call 
(Calling external payment API for $150.00...)
.../test_commerce_marked.py:32: DeprecationWarning: This payment gateway is deprecated.
  result = process_payment_via_api(total=150.00)
(API call successful.)
PASSED [100%]

================== 2 passed, 1 skipped, 1 xfailed in 2.04s =================
```

**Expected vs. Actual Improvement:** The test is now marked as `SKIPPED`, and the reason is displayed. Notice the total test time dropped by about one second—the time it took to `connect_to_db()`. We've made our test run faster by not executing irrelevant tests.

### Iteration 3: Suppressing Warnings with `@pytest.mark.filterwarnings`

Our final problem is the `DeprecationWarning` from the payment API call. While it's good to be aware of these, sometimes they are out of our control and just add noise.

**Current Limitation:** A test triggers a warning that we can't fix immediately, cluttering the test output.

**What we need:** A way to tell pytest: "For this specific test, I know about this specific warning, and I want you to ignore it."

### Technique Introduced: `@pytest.mark.filterwarnings`

This marker allows you to add a warning filter for a specific test, using the same syntax as Python's own warning filters.

### Solution Implementation

Let's suppress the `DeprecationWarning` in `test_payment_api_call`.

**Before:**
```python
# A very slow API test
def test_payment_api_call():
    """Tests the external payment API call."""
    result = process_payment_via_api(total=150.00)
    assert result["status"] == "paid"
    assert result["amount"] == 150.00
```

**After (`test_commerce_marked.py`):**
```python
# ... other tests ...

# A very slow API test
@pytest.mark.filterwarnings("ignore:This payment gateway is deprecated.")
def test_payment_api_call():
    """Tests the external payment API call."""
    result = process_payment_via_api(total=150.00)
    assert result["status"] == "paid"
    assert result["amount"] == 150.00
```

### Verification

```bash
$ pytest -v test_commerce_marked.py
=========================== test session starts ============================
...
collected 4 items

test_commerce_marked.py::test_order_total PASSED                      [ 25%]
test_commerce_marked.py::test_order_discount_logic XFAIL              [ 50%]
test_commerce_marked.py::test_database_connection SKIPPED (Database schema is being refactored.) [ 75%]
test_commerce_marked.py::test_payment_api_call 
(Calling external payment API for $150.00...)
(API call successful.)
PASSED [100%]

================== 2 passed, 1 skipped, 1 xfailed in 2.04s =================
```

**Expected vs. Actual Improvement:** The test output is now clean. The `DeprecationWarning` is no longer displayed in the summary, making the output easier to read. We've successfully managed all the initial problems using built-in markers.

**Limitation Preview:** This is great, but we still haven't solved our biggest organizational problem: we can't selectively run tests based on their *type* (e.g., fast unit tests vs. slow API tests). For that, we need to create our own markers.

## Creating Custom Markers

## Creating Custom Markers

The true power of markers comes from creating your own. You can define any marker you want to categorize your tests by functionality, speed, or any other criteria relevant to your project.

### Iteration 4: Categorizing Tests with Custom Markers

Our test suite has different kinds of tests:
- `test_order_total`: A fast unit test.
- `test_order_discount_logic`: Another fast unit test.
- `test_database_connection`: A slower integration test.
- `test_payment_api_call`: A very slow test that depends on an external API.

**Current Limitation:** We have no way to distinguish between these types of tests. `pytest` treats them all the same, forcing us to run them all, every time.

**New Scenario:** As a developer, I want to run only the fast `unit` tests while I'm actively coding to get quick feedback.

### Technique Introduced: Custom Marker Naming

Creating a custom marker is as simple as adding a decorator with a name you choose. The syntax is `@pytest.mark.your_marker_name`.

Let's categorize our tests using the markers `unit`, `integration`, and `api`.

### Solution Implementation

We will apply these new markers to our `test_commerce_marked.py` file.

**Before:**
The tests had only built-in markers (`xfail`, `skip`, etc.) or no markers at all.

**After (`test_commerce_marked.py`):**
```python
import pytest
from commerce_system import Order, Product, connect_to_db, process_payment_via_api

# A fast, simple unit test
@pytest.mark.unit
def test_order_total():
    """Tests the total calculation for a simple order."""
    products = [Product("Laptop", 1200.00), Product("Mouse", 25.00)]
    order = Order(products)
    assert order.total() == 1225.00

# A test for a known bug
@pytest.mark.unit
@pytest.mark.xfail(reason="Discount logic is buggy, see TICKET-123")
def test_order_discount_logic():
    """Tests the discount application logic, which is known to be buggy."""
    products = [Product("Keyboard", 100.00), Product("Mouse", 25.00)]
    order = Order(products)
    order.apply_discount(10)
    assert order.total() == 112.50

# A slower integration test
@pytest.mark.integration
@pytest.mark.skip(reason="Database schema is being refactored.")
def test_database_connection():
    """Tests the function that connects to the database."""
    conn = connect_to_db()
    assert conn["status"] == "connected"

# A very slow API test
@pytest.mark.api
@pytest.mark.filterwarnings("ignore:This payment gateway is deprecated.")
def test_payment_api_call():
    """Tests the external payment API call."""
    result = process_payment_via_api(total=150.00)
    assert result["status"] == "paid"
    assert result["amount"] == 150.00
```
Notice that you can stack markers. `test_order_discount_logic` is now marked as both `unit` and `xfail`.

### Verification

Let's run pytest and see what happens.

```bash
$ pytest -v test_commerce_marked.py
=========================== test session starts ============================
...
collected 4 items

test_commerce_marked.py::test_order_total PASSED                      [ 25%]
test_commerce_marked.py::test_order_discount_logic XFAIL              [ 50%]
test_commerce_marked.py::test_database_connection SKIPPED (Database schema is being refactored.) [ 75%]
test_commerce_marked.py::test_payment_api_call PASSED                 [100%]
=========================== warnings summary ===============================
.../test_commerce_marked.py:7
  .../test_commerce_marked.py:7: PytestUnknownMarkWarning: Unknown pytest.mark.unit - is this a typo?  You can register custom marks to avoid this warning.
    @pytest.mark.unit

.../test_commerce_marked.py:22
  .../test_commerce_marked.py:22: PytestUnknownMarkWarning: Unknown pytest.mark.integration - is this a typo?  You can register custom marks to avoid this warning.
    @pytest.mark.integration

.../test_commerce_marked.py:30
  .../test_commerce_marked.py:30: PytestUnknownMarkWarning: Unknown pytest.mark.api - is this a typo?  You can register custom marks to avoid this warning.
    @pytest.mark.api

-- Docs: https://docs.pytest.org/en/stable/how-to/mark.html#registering-marks
================== 2 passed, 1 skipped, 1 xfailed, 3 warnings in 2.05s =================
```

The tests ran as before, but now we have a new problem: `PytestUnknownMarkWarning`.

## Registering Markers in Configuration

## Registering Markers in Configuration

We've successfully added custom markers, but pytest is helpfully warning us that it doesn't recognize them. This is a safety feature to prevent typos (e.g., `@pytest.mark.integraton` instead of `@pytest.mark.integration`). To make our markers "official," we need to register them.

### Iteration 5: Eliminating "Unknown Marker" Warnings

**Current Limitation:** Our test suite runs but produces `PytestUnknownMarkWarning` for each of our custom markers, cluttering the output and indicating poor practice.

### Diagnostic Analysis: Reading the Warning

**The complete output**:
```bash
=========================== warnings summary ===============================
.../test_commerce_marked.py:7
  .../test_commerce_marked.py:7: PytestUnknownMarkWarning: Unknown pytest.mark.unit - is this a typo?  You can register custom marks to avoid this warning.
    @pytest.mark.unit
...
```

**Let's parse this section by section**:

1.  **The warning type**: `PytestUnknownMarkWarning`.
    -   What this tells us: This is a specific warning from pytest itself about the usage of markers.

2.  **The message**: `Unknown pytest.mark.unit - is this a typo?`
    -   What this tells us: Pytest is explicitly asking if we made a mistake. It doesn't know if `unit` is a valid category we intended to create or a typo of a built-in marker.

3.  **The suggestion**: `You can register custom marks to avoid this warning.`
    -   What this tells us: Pytest is guiding us directly to the solution.

**Root cause identified**: We have defined custom markers in our test code, but we haven't declared them in our project's configuration.
**What we need**: A configuration file (`pytest.ini`, `pyproject.toml`, or `tox.ini`) where we can list our custom markers.

### Technique Introduced: The `markers` Configuration Key

You can register markers by creating a `pytest.ini` file in the root of your project and adding a `[pytest]` section with a `markers` key.

### Solution Implementation

Let's create a `pytest.ini` file.

**`pytest.ini`:**

```ini
[pytest]
markers =
    unit: marks tests as unit tests (fast, no dependencies)
    integration: marks tests as integration tests (slower, may require services like a DB)
    api: marks tests as API tests (very slow, requires network access)
```

This format is simple: each line under `markers` is `marker_name: description`. The description is important, as it will appear in pytest's help text.

### Verification

First, let's see if pytest recognizes our markers now by running `pytest --markers`.

```bash
$ pytest --markers

@pytest.mark.unit: marks tests as unit tests (fast, no dependencies)
@pytest.mark.integration: marks tests as integration tests (slower, may require services like a DB)
@pytest.mark.api: marks tests as API tests (very slow, requires network access)

@pytest.mark.filterwarnings(warning): add a warning filter to the given test.
@pytest.mark.skip(reason=None): skip the given test function with an optional reason.
@pytest.mark.skipif(condition, ..., reason=None): skip the given test function if any of the conditions evaluate to true.
@pytest.mark.xfail(condition, ..., reason=None, run=True, raises=None, strict=False): mark the test function as an expected failure if any of the conditions are true.
@pytest.mark.parametrize(argnames, argvalues, ...): call a test function multiple times with different arguments.
```

Success! Our custom markers `unit`, `integration`, and `api` are listed alongside the built-in ones, complete with our descriptions.

Now, let's re-run our test suite and check for warnings.

```bash
$ pytest -v test_commerce_marked.py
=========================== test session starts ============================
...
collected 4 items

test_commerce_marked.py::test_order_total PASSED                      [ 25%]
test_commerce_marked.py::test_order_discount_logic XFAIL              [ 50%]
test_commerce_marked.py::test_database_connection SKIPPED (Database schema is being refactored.) [ 75%]
test_commerce_marked.py::test_payment_api_call PASSED                 [100%]

================== 2 passed, 1 skipped, 1 xfailed in 2.06s =================
```

**Expected vs. Actual Improvement:** The output is perfectly clean. The `PytestUnknownMarkWarning` messages are gone. Our test suite is now well-organized *and* correctly configured.

**Limitation Preview:** We've done all the setup. We've categorized and registered our markers. But we still haven't actually *used* them to change which tests are run. That's the final, crucial step.

## Filtering Tests by Markers

## Filtering Tests by Markers

This is where all our organizational work pays off. With markers in place, we can use the `-m` command-line option to run specific subsets of our tests.

### Iteration 6: Selectively Running Tests with `-m`

**Current Limitation:** We have markers, but `pytest` still runs everything it finds (respecting `skip` and `xfail`, of course). We cannot yet run *only* the unit tests.

### Technique Introduced: The `-m` Command-Line Option

The `-m` flag allows you to specify a "marker expression" to select tests.

### Solution Implementation & Verification

Let's try out some common filtering scenarios.

**1. Run only the unit tests:**
This is our primary goal: get fast feedback during development.

```bash
$ pytest -v -m unit
=========================== test session starts ============================
...
collected 4 items / 2 deselected / 2 selected

test_commerce_marked.py::test_order_total PASSED                      [ 50%]
test_commerce_marked.py::test_order_discount_logic XFAIL              [100%]

==================== 1 passed, 1 xfailed, 2 deselected in 0.02s ====================
```

Look at that! Pytest selected only the two tests marked `unit`. The integration and API tests were `deselected`. The total run time was a fraction of a second. This is the rapid feedback loop we wanted.

**2. Run only the integration tests:**

```bash
$ pytest -v -m integration
=========================== test session starts ============================
...
collected 4 items / 3 deselected / 1 selected

test_commerce_marked.py::test_database_connection SKIPPED (Database schema is being refactored.) [100%]

=================== 1 skipped, 3 deselected in 0.01s ===================
```

As expected, it selected our single integration test, which is currently skipped.

**3. Run everything *except* the slow API tests:**
You can use boolean logic. `not` is very useful for excluding slow or disruptive tests.

```bash
$ pytest -v -m "not api"
=========================== test session starts ============================
...
collected 4 items / 1 deselected / 3 selected

test_commerce_marked.py::test_order_total PASSED                      [ 33%]
test_commerce_marked.py::test_order_discount_logic XFAIL              [ 66%]
test_commerce_marked.py::test_database_connection SKIPPED (Database schema is being refactored.) [100%]

================ 1 passed, 1 skipped, 1 xfailed, 1 deselected in 0.02s ================
```

This ran the `unit` and `integration` tests but skipped the `api` test. This is perfect for a pre-commit hook or a quick CI check.

**4. Run unit tests OR integration tests:**

```bash
$ pytest -v -m "unit or integration"
=========================== test session starts ============================
...
collected 4 items / 1 deselected / 3 selected

test_commerce_marked.py::test_order_total PASSED                      [ 33%]
test_commerce_marked.py::test_order_discount_logic XFAIL              [ 66%]
test_commerce_marked.py::test_database_connection SKIPPED (Database schema is being refactored.) [100%]

================ 1 passed, 1 skipped, 1 xfailed, 1 deselected in 0.02s ================
```

The expression `unit or integration` selects any test that has at least one of those markers. You can also use `and` to select tests that must have *all* specified markers. For example, `pytest -m "slow and database"` would run tests marked with both `@pytest.mark.slow` and `@pytest.mark.database`.

We have now achieved full control over which tests we run and when.

## Organizing Tests by Category

## Organizing Tests by Category

We've completed the journey from a disorganized, all-or-nothing test suite to a flexible, well-managed collection of tests that can be run selectively based on our needs.

### The Journey: From Problem to Solution

| Iteration | Failure Mode / Problem                               | Technique Applied                  | Result                                                              |
| --------- | ---------------------------------------------------- | ---------------------------------- | ------------------------------------------------------------------- |
| 0         | One failing test breaks the build.                   | None                               | Initial state: `1 failed, 3 passed`.                                |
| 1         | Known bug causes CI failure.                         | `@pytest.mark.xfail`               | Test is run but its failure is expected; build passes.              |
| 2         | Test for unavailable feature slows down suite.       | `@pytest.mark.skip`                | Test is not executed at all, speeding up the run.                   |
| 3         | Unactionable warnings clutter output.                | `@pytest.mark.filterwarnings`      | Specific warnings are suppressed for a clean test report.           |
| 4         | No way to differentiate test types.                  | Custom markers (`@pytest.mark.unit`) | Tests are categorized with metadata.                                |
| 5         | Custom markers produce `PytestUnknownMarkWarning`.   | Registering markers in `pytest.ini`  | Warnings are eliminated; markers are officially part of the project. |
| 6         | Still running all tests by default.                  | Command-line flag `-m`             | Full control to run specific subsets of tests (e.g., `pytest -m unit`). |

### Final Implementation

Here is the complete, production-ready version of our test file and configuration.

**`pytest.ini`**

```ini
[pytest]
markers =
    unit: marks tests as unit tests (fast, no dependencies)
    integration: marks tests as integration tests (slower, may require services like a DB)
    api: marks tests as API tests (very slow, requires network access)
```

**`test_commerce_final.py`**

```python
import pytest
from commerce_system import Order, Product, connect_to_db, process_payment_via_api

@pytest.mark.unit
def test_order_total():
    """Tests the total calculation for a simple order."""
    products = [Product("Laptop", 1200.00), Product("Mouse", 25.00)]
    order = Order(products)
    assert order.total() == 1225.00

@pytest.mark.unit
@pytest.mark.xfail(reason="Discount logic is buggy, see TICKET-123")
def test_order_discount_logic():
    """Tests the discount application logic, which is known to be buggy."""
    products = [Product("Keyboard", 100.00), Product("Mouse", 25.00)]
    order = Order(products)
    order.apply_discount(10)
    assert order.total() == 112.50

@pytest.mark.integration
@pytest.mark.skip(reason="Database schema is being refactored.")
def test_database_connection():
    """Tests the function that connects to the database."""
    conn = connect_to_db()
    assert conn["status"] == "connected"

@pytest.mark.api
@pytest.mark.filterwarnings("ignore:This payment gateway is deprecated.")
def test_payment_api_call():
    """Tests the external payment API call."""
    result = process_payment_via_api(total=150.00)
    assert result["status"] == "paid"
    assert result["amount"] == 150.00
```

### Decision Framework: Which Approach When?

Markers are a tool for organization. Here's a guide on when and how to use them effectively.

| Use Case                               | Recommended Marker Strategy                                                              | Example                                                              |
| -------------------------------------- | ---------------------------------------------------------------------------------------- | -------------------------------------------------------------------- |
| **Managing known bugs**                | Use `@pytest.mark.xfail` with a clear `reason`.                                          | `@pytest.mark.xfail(reason="Bug #451: rounding error")`              |
| **Temporarily disabling tests**        | Use `@pytest.mark.skip` for features in flux or broken environments.                     | `@pytest.mark.skip(reason="Feature X under rework")`                 |
| **Categorizing by test scope/speed**   | Create custom markers like `unit`, `integration`, `e2e`.                                 | `@pytest.mark.unit`, `@pytest.mark.integration`                      |
| **Categorizing by feature area**       | Create custom markers for major parts of your app, like `auth`, `payments`, `search`.    | `@pytest.mark.payments`                                              |
| **Marking tests with special needs**   | Create markers for tests that need specific resources, like `needs_db` or `needs_network`. | `@pytest.mark.needs_db`                                              |
| **Running a test with many inputs**    | **Do not use markers.** Use `@pytest.mark.parametrize` instead (see Chapter 7).          | `@pytest.mark.parametrize("input,expected", [...])`                  |

### Lessons Learned

Markers embody the pytest philosophy of **explicit over implicit**. By tagging a test, you are explicitly stating its characteristics and purpose beyond just its name.

1.  **Markers Separate *What* from *When***: The test function body defines *what* is being tested. The markers define *when* and *how* that test should be run. This separation of concerns is key to a maintainable test suite.
2.  **Configuration is Code**: Registering your markers in `pytest.ini` is as important as writing the tests themselves. It makes your test suite self-documenting and prevents errors.
3.  **Empower Your CI/CD**: A well-marked test suite allows you to create sophisticated CI/CD pipelines. You can have a fast pipeline that runs only `unit` tests on every commit, and a slower, more comprehensive pipeline that runs `"not e2e"` tests on every pull request, and a full nightly build that runs everything. This granular control is essential for scaling your testing efforts.
