# Chapter 2: Understanding Test Fundamentals

## Assertions: The Heart of Testing

## Assertions: The Heart of Testing

At its core, every test performs a simple, crucial task: it checks if something is true. In programming, we call this check an **assertion**. An assertion is a statement that declares a condition we expect to be true at a specific point in our code's execution. If the condition is true, the test passes silently. If it's false, the test fails loudly, stopping execution and telling us exactly what went wrong.

Pytest uses Python's built-in `assert` statement, enhancing it with powerful introspection to provide detailed failure messages. This makes your tests clean, readable, and incredibly informative when they fail.

### Phase 1: Establish the Reference Implementation

Let's anchor our learning in a concrete example. Imagine we're building an e-commerce platform. A key piece of business logic is calculating customer discounts. Here is our initial function, which lives in a file named `pricing.py`.

**Anchor Example**: `calculate_discount()`

This function is supposed to give a 10% discount on orders of $100 or more, and a 5% discount on orders between $20 and $99.99.

```python
# src/ecommerce/pricing.py

def calculate_discount(order_total: float) -> float:
    """
    Calculates a discount based on the order total.
    - 10% for orders >= $100
    - 5% for orders >= $20
    """
    if order_total >= 100:
        return order_total * 0.10
    elif order_total > 20:  # A subtle bug is lurking here!
        return order_total * 0.05
    return 0.0
```

Now, let's write our first test to verify the 10% discount for a high-value order. We'll create a `tests` directory and place our test file there.

```python
# tests/test_pricing.py

from src.ecommerce.pricing import calculate_discount

def test_calculate_discount_for_high_value_order():
    """
    Tests that a 10% discount is applied for orders of $100 or more.
    """
    # Arrange: Set up the test data
    order_total = 200.0

    # Act: Call the function we are testing
    discount = calculate_discount(order_total)

    # Assert: Check if the result is what we expect
    assert discount == 20.0
```

This test follows the classic "Arrange-Act-Assert" pattern:
1.  **Arrange**: We set up the necessary inputs. Here, an `order_total` of 200.0.
2.  **Act**: We execute the code under test. We call `calculate_discount()`.
3.  **Assert**: We check the outcome. We assert that the `discount` is equal to 20.0.

Let's run this test. It should pass, as our function's logic for orders over $100 seems correct.

```bash
$ pytest
========================= test session starts ==========================
...
collected 1 item

tests/test_pricing.py .                                          [100%]

========================== 1 passed in ...s ==========================
```

### Iteration 1: Exposing a Bug with a New Assertion

Our first test passed, but does that mean the function is perfect? A single test case rarely covers all business logic. Let's add a test for the 5% discount tier. According to the rules, an order of exactly $20.00 should receive a 5% discount, which is $1.00.

Here's the new test added to our file:

```python
# tests/test_pricing.py (updated)

from src.ecommerce.pricing import calculate_discount

def test_calculate_discount_for_high_value_order():
    """
    Tests that a 10% discount is applied for orders of $100 or more.
    """
    assert calculate_discount(200.0) == 20.0

def test_calculate_discount_at_lower_boundary():
    """
    Tests that a 5% discount is applied for an order of exactly $20.
    """
    # Arrange
    order_total = 20.0
    expected_discount = 1.0

    # Act
    actual_discount = calculate_discount(order_total)

    # Assert
    assert actual_discount == expected_discount
```

Now, let's run pytest again and see what happens.

```bash
$ pytest
========================= test session starts ==========================
...
collected 2 items

tests/test_pricing.py .F                                         [100%]

=============================== FAILURES ===============================
___________ test_calculate_discount_at_lower_boundary ____________

    def test_calculate_discount_at_lower_boundary():
        """
        Tests that a 5% discount is applied for an order of exactly $20.
        """
        # Arrange
        order_total = 20.0
        expected_discount = 1.0

        # Act
        actual_discount = calculate_discount(order_total)

        # Assert
>       assert actual_discount == expected_discount
E       assert 0.0 == 1.0

tests/test_pricing.py:22: AssertionError
======================= short test summary info ========================
FAILED tests/test_pricing.py::test_calculate_discount_at_lower_boundary - assert 0.0 == 1.0
===================== 1 failed, 1 passed in ...s =====================
```

### Diagnostic Analysis: Reading the Failure

This output is a goldmine of information. Let's treat it as data, not a judgment.

**The complete output**:
```bash
$ pytest
========================= test session starts ==========================
...
collected 2 items

tests/test_pricing.py .F                                         [100%]

=============================== FAILURES ===============================
___________ test_calculate_discount_at_lower_boundary ____________

    def test_calculate_discount_at_lower_boundary():
        """
        Tests that a 5% discount is applied for an order of exactly $20.
        """
        # Arrange
        order_total = 20.0
        expected_discount = 1.0

        # Act
        actual_discount = calculate_discount(order_total)

        # Assert
>       assert actual_discount == expected_discount
E       assert 0.0 == 1.0

tests/test_pricing.py:22: AssertionError
======================= short test summary info ========================
FAILED tests/test_pricing.py::test_calculate_discount_at_lower_boundary - assert 0.0 == 1.0
===================== 1 failed, 1 passed in ...s =====================
```

**Let's parse this section by section**:

1.  **The summary line**: `FAILED tests/test_pricing.py::test_calculate_discount_at_lower_boundary - assert 0.0 == 1.0`
    -   **What this tells us**: The exact test function that failed (`test_calculate_discount_at_lower_boundary` in the file `tests/test_pricing.py`) and a summary of the assertion that broke.

2.  **The traceback**:
    ```python
    >       assert actual_discount == expected_discount
    E       assert 0.0 == 1.0

    tests/test_pricing.py:22: AssertionError
    ```
    -   **What this tells us**: The failure happened on line 22 of `tests/test_pricing.py`. The `>` points to the exact line of code.
    -   **Key line**: The line starting with `E` (for Error) shows the values that were being compared when the assertion failed.

3.  **The assertion introspection**:
    ```
    assert 0.0 == 1.0
    ```
    -   **What this tells us**: This is pytest's magic. It didn't just tell us the assertion failed; it inspected the variables in the `assert` statement and showed us their values. We were expecting `actual_discount` to be `1.0`, but it was actually `0.0`. We don't need to add `print()` statements or use a debugger to see what went wrong.

**Root cause identified**: The function returned a discount of `0.0` for an order of `20.0`, when we expected `1.0`.
**Why the current approach can't solve this**: Looking back at the source code, the bug is clear: `elif order_total > 20:`. This condition is not met for an order of exactly 20. It should be `order_total >= 20`.
**What we need**: A fix in the business logic to correctly handle the boundary condition.

### The Fix and Verification

Let's correct the bug in `src/ecommerce/pricing.py`.

**Before**:

```python
# src/ecommerce/pricing.py (buggy version)

def calculate_discount(order_total: float) -> float:
    ...
    elif order_total > 20:  # The bug is here
        return order_total * 0.05
    ...
```

**After**:

```python
# src/ecommerce/pricing.py (fixed version)

def calculate_discount(order_total: float) -> float:
    """
    Calculates a discount based on the order total.
    - 10% for orders >= $100
    - 5% for orders >= $20
    """
    if order_total >= 100:
        return order_total * 0.10
    elif order_total >= 20:  # The fix is here
        return order_total * 0.05
    return 0.0
```

Now, we re-run our tests to verify the fix.

```bash
$ pytest
========================= test session starts ==========================
...
collected 2 items

tests/test_pricing.py ..                                         [100%]

========================== 2 passed in ...s ==========================
```

Success! The failing test now passes, and our first test still passes, giving us confidence that our fix didn't break existing functionality. This is the fundamental cycle of testing: write a test that captures a requirement, see it fail, write the code to make it pass, and repeat. The `assert` statement is the gatekeeper of this entire process.

## Test Functions vs. Other Functions

## Test Functions vs. Other Functions

In our test file, we have functions like `test_calculate_discount...`. These look like normal Python functions, but they are special. A **test function** is a function that pytest recognizes and executes as a test case. A regular Python function, by contrast, executes business logic or acts as a helper.

What makes a function a *test function* in the eyes of pytest? It's all about **convention**.

Let's compare the function we are testing with the function that tests it.

**Function Under Test (in `src/ecommerce/pricing.py`)**:

```python
def calculate_discount(order_total: float) -> float:
    # ... business logic ...
    return discount
```

**Test Function (in `tests/test_pricing.py`)**:

```python
def test_calculate_discount_at_lower_boundary():
    # ... arrange, act, assert ...
    assert actual_discount == expected_discount
```

### Key Differences

1.  **Purpose**:
    *   `calculate_discount`: Performs a business task. It takes data, processes it, and **returns a value**. This is production code.
    *   `test_calculate_discount...`: Verifies behavior. It sets up a scenario, calls the production code, and **makes an assertion**. It typically **does not return a value**. Its success or failure is communicated by passing the assertion or raising an `AssertionError`.

2.  **Naming**:
    *   `calculate_discount`: Named for what it *does*.
    *   `test_calculate_discount...`: Must be prefixed with `test_`. This is the primary signal pytest uses to identify it as a test.

3.  **Execution**:
    *   `calculate_discount`: Called by your application's logic.
    *   `test_calculate_discount...`: Called automatically by the pytest test runner. You almost never call a test function directly yourself.

### Helper Functions in Test Files

You can, and should, have other functions in your test files that are *not* tests. These are often called "helper functions." They exist to reduce code duplication and make your tests more readable.

Imagine we need to create complex order objects for our tests. We could write a helper.

```python
# tests/test_pricing.py (with a helper)

from src.ecommerce.pricing import calculate_discount

# This is a HELPER function, not a test function.
# Pytest will ignore it because its name doesn't start with "test_".
def create_order_with_total(total: float) -> dict:
    """A helper to create a sample order dictionary."""
    return {"customer_id": 123, "total": total, "items": []}

def test_calculate_discount_for_high_value_order():
    order = create_order_with_total(200.0)
    assert calculate_discount(order["total"]) == 20.0

def test_calculate_discount_at_lower_boundary():
    order = create_order_with_total(20.0)
    assert calculate_discount(order["total"]) == 1.0
```

In this example, `create_order_with_total` is a regular Python function. Pytest ignores it during test collection because its name doesn't start with `test_`. It simply serves to make the test functions cleaner.

This distinction is fundamental to the next topic: how pytest knows which functions to run in the first place.

## Test Discovery: How Pytest Finds Your Tests

## Test Discovery: How Pytest Finds Your Tests

When you type `pytest` in your terminal, it springs into action, searching your project for tests to run. This process is called **test discovery**. It's not magic; it's a simple set of rules and conventions. If you don't follow them, pytest won't find your tests.

### Iteration 2: Breaking Discovery

Let's see this in action by deliberately breaking the convention. We'll take one of our working tests and rename it so it no longer starts with `test_`.

**Current State**: Our `tests/test_pricing.py` file has two test functions that are correctly discovered and run.

**Limitation**: Our understanding of *why* they are discovered is based on an assumption. Let's prove it.

**New Scenario**: What happens if we rename `test_calculate_discount_at_lower_boundary` to `check_discount_at_lower_boundary`?

```python
# tests/test_pricing.py (with renamed function)

from src.ecommerce.pricing import calculate_discount

def test_calculate_discount_for_high_value_order():
    assert calculate_discount(200.0) == 20.0

# This function will NOT be discovered by pytest
def check_discount_at_lower_boundary():
    assert calculate_discount(20.0) == 1.0
```

Now, let's run pytest. Pay close attention to the output.

```bash
$ pytest
========================= test session starts ==========================
...
collected 1 item

tests/test_pricing.py .                                          [100%]

========================== 1 passed in ...s ==========================
```

### Diagnostic Analysis: Reading the "Failure"

The tests didn't fail, but something is wrong. This is a different kind of failure: a failure of the test suite itself.

**The complete output**:
```bash
$ pytest
========================= test session starts ==========================
...
collected 1 item

tests/test_pricing.py .                                          [100%]

========================== 1 passed in ...s ==========================
```

**Let's parse this**:

1.  **The summary line**: `collected 1 item`
    -   **What this tells us**: This is the most important clue. Previously, pytest collected 2 items. Now it only finds one. This means one of our tests has become invisible to the test runner.

2.  **The test execution line**: `tests/test_pricing.py .`
    -   **What this tells us**: Pytest ran the tests in `test_pricing.py` and found one test, which passed (indicated by the `.`).

**Root cause identified**: Pytest did not run our second test because its name, `check_discount_at_lower_boundary`, does not follow the required naming convention.
**Why the current approach can't solve this**: The function is perfectly valid Python code, but it doesn't conform to the contract pytest expects for a test function.
**What we need**: To understand and adhere to pytest's discovery rules.

### Pytest's Discovery Rules

Pytest searches the current directory and subdirectories for test modules and functions based on these default conventions:

1.  **File Names**: It looks for files named `test_*.py` or `*_test.py`.
2.  **Function Names**: Inside those files, it looks for functions prefixed with `test_`.
3.  **Class Names**: It will also discover tests inside classes prefixed with `Test` (e.g., `class TestPricing:`), as long as the methods inside are also prefixed with `test_`. We will cover test classes in a later chapter.

Our function `check_discount_at_lower_boundary` failed rule #2.

### Banish Magic with Mechanics: `pytest --collect-only`

How can you see what pytest sees without actually running the tests? Pytest provides a powerful command-line flag for this: `--collect-only`. This command performs the entire discovery process and then prints a list of all the tests it found.

Let's run it on our broken version:

```bash
$ pytest --collect-only
========================= test session starts ==========================
...
collected 1 item
<Module tests/test_pricing.py>
  <Function test_calculate_discount_for_high_value_order>

======================= 1 item collected in ...s =======================
```

This output is crystal clear. Pytest reports that it found the module `tests/test_pricing.py` and, inside it, only one test function: `test_calculate_discount_for_high_value_order`. Our other function is completely absent from this list.

### The Fix and Verification

Let's fix the problem by renaming the function back to its correct form.

**Before**:
`def check_discount_at_lower_boundary():`

**After**:
`def test_calculate_discount_at_lower_boundary():`

Now, let's run `pytest --collect-only` again.

```bash
$ pytest --collect-only
========================= test session starts ==========================
...
collected 2 items
<Module tests/test_pricing.py>
  <Function test_calculate_discount_for_high_value_order>
  <Function test_calculate_discount_at_lower_boundary>

======================= 2 items collected in ...s ======================
```

As expected, pytest now sees both of our tests. Running `pytest` will now execute both, and they will both pass. Adhering to the simple `test_` naming convention is the key that unlocks all of pytest's power.

## Naming Conventions That Matter

## Naming Conventions That Matter

We've established that test functions must start with `test_`. But what comes after the prefix is just as important. A well-named test serves as living documentation for your codebase. When a test fails a year from now, a descriptive name can save you hours of debugging.

Consider these two test names for the same test:

1.  `test_function_1()`
2.  `test_calculate_discount_for_order_total_of_20_is_5_percent()`

If `test_function_1()` fails, the output tells you almost nothing. You have to read the test's code to understand what it was trying to do. If the second test fails, you know the *exact business rule* that was violated before you even look at the code.

### A Practical Naming Pattern: `test_when_..._should_...`

A highly effective and readable convention for naming tests is the "When/Should" or "Given/When/Then" pattern.

The structure is: `test_when_[Action or State]_should_[Expected Outcome]`

Let's refactor our existing test names to follow this pattern.

```python
# tests/test_pricing.py (with improved names)

from src.ecommerce.pricing import calculate_discount

def test_when_order_total_is_high_should_apply_10_percent_discount():
    """
    Given: An order total of $200
    When: The discount is calculated
    Then: The discount should be 10% ($20)
    """
    assert calculate_discount(200.0) == 20.0

def test_when_order_total_is_at_5_percent_boundary_should_apply_discount():
    """
    Given: An order total of exactly $20
    When: The discount is calculated
    Then: The discount should be 5% ($1)
    """
    assert calculate_discount(20.0) == 1.0
```

### Verbose Names Make for Readable Reports

This might seem overly verbose, but the payoff comes when you run your tests. By default, pytest's output is compact. However, using the verbose flag (`-v`), pytest will print each test's full name next to its result.

```bash
$ pytest -v
========================= test session starts ==========================
...
collected 2 items

tests/test_pricing.py::test_when_order_total_is_high_should_apply_10_percent_discount PASSED [ 50%]
tests/test_pricing.py::test_when_order_total_is_at_5_percent_boundary_should_apply_discount PASSED [100%]

========================== 2 passed in ...s ==========================
```

This output reads like a checklist of your system's features. Each line is a verifiable statement about how your code behaves. When a line item turns from `PASSED` to `FAILED`, you know precisely which feature broke.

Good test names are one of the highest-leverage, lowest-effort practices you can adopt for a maintainable codebase.

## Organizing Tests in Your Project

## Organizing Tests in Your Project

As your project grows, you'll have dozens or even hundreds of test files. A logical and consistent project structure is essential for keeping your test suite manageable.

There are two common patterns for organizing tests in a Python project.

### Pattern 1: The `tests` Directory (Recommended)

This is the most common and recommended structure. You create a top-level `tests` directory alongside your source code directory (often called `src` or named after your project).

```
my_ecommerce_project/
├── .gitignore
├── pyproject.toml
├── src/
│   └── ecommerce/
│       ├── __init__.py
│       └── pricing.py      # <-- Code under test
└── tests/
    ├── __init__.py
    └── test_pricing.py     # <-- Our test file
```

**Advantages**:

*   **Clear Separation**: Production code and test code are completely separate. This prevents you from accidentally packaging your tests into your final application distribution.
*   **Easy to Run**: You can run all tests by simply typing `pytest` in the project root. Pytest will automatically discover the `tests` directory.
*   **Mirrors Structure**: The structure inside `tests` can mirror the structure of your `src` directory, making it easy to find the tests for a specific module (e.g., `tests/api/test_client.py` tests `src/ecommerce/api/client.py`).

### Pattern 2: Tests Alongside Code

In this pattern, test files are placed directly inside your source code packages.

```
my_ecommerce_project/
├── .gitignore
├── pyproject.toml
└── src/
    └── ecommerce/
        ├── __init__.py
        ├── pricing.py
        └── test_pricing.py  # <-- Test file next to the code
```

**Advantages**:

*   **Proximity**: Tests are located right next to the code they are testing, which can be convenient for small modules.

**Disadvantages**:

*   **Packaging Complexity**: You need to configure your build system (e.g., in `pyproject.toml`) to exclude the `test_*.py` files from your final production package. This is an extra step that is easy to get wrong.
*   **Clutter**: It can clutter your source directories, mixing production logic with test logic.

For these reasons, **we will use and recommend the `tests` directory structure throughout this book.** It is the standard for the vast majority of modern Python projects.

### The Role of `__init__.py`

You'll notice empty `__init__.py` files in both the `src/ecommerce` and `tests` directories. These files tell Python to treat the directories as "packages." This allows you to use Python's import system cleanly. For example, it's what allows `from src.ecommerce.pricing import calculate_discount` to work correctly when you run `pytest` from the project root. Even if the files are empty, their presence is important.

## Running Specific Tests

## Running Specific Tests

As your test suite grows, running every single test every time you make a small change can become slow and inefficient. Pytest provides several powerful ways to run only the tests you care about, which is crucial for a fast and productive development workflow.

Let's expand our project slightly to demonstrate these features.

```
tests/
├── __init__.py
├── test_inventory.py   # <-- New file
└── test_pricing.py
```

Our new `test_inventory.py` file contains a simple test.

```python
# tests/test_inventory.py

def test_stock_level_is_reduced_on_purchase():
    # A placeholder test for demonstration
    assert True

def test_cannot_purchase_out_of_stock_item():
    # A placeholder test for demonstration
    assert True
```

Now, if we run `pytest --collect-only` from the root, we see all four tests.

```bash
$ pytest --collect-only
========================= test session starts ==========================
...
collected 4 items
<Module tests/test_inventory.py>
  <Function test_stock_level_is_reduced_on_purchase>
  <Function test_cannot_purchase_out_of_stock_item>
<Module tests/test_pricing.py>
  <Function test_when_order_total_is_high_should_apply_10_percent_discount>
  <Function test_when_order_total_is_at_5_percent_boundary_should_apply_discount>

======================= 4 items collected in ...s ======================
```

### Running Tests by File or Directory

The simplest way to select tests is by providing a path.

**To run all tests in a directory**:

```bash
# This will run all tests inside the tests/ directory (which is the default anyway)
$ pytest tests/

# Output will show all 4 tests running
...
========================== 4 passed in ...s ==========================
```

**To run all tests in a single file**:

```bash
# This will only run the 2 tests inside test_pricing.py
$ pytest tests/test_pricing.py

# Output will show only 2 tests running
...
========================== 2 passed in ...s ==========================
```

### Running a Single Test by Node ID

If you want to run one specific test function, you can specify its "Node ID". The Node ID is the unique path to a test, in the format `path/to/file.py::test_function_name`.

**To run one specific test function**:

```bash
$ pytest tests/test_pricing.py::test_when_order_total_is_high_should_apply_10_percent_discount

# Output will show only 1 test running
...
========================== 1 passed in ...s ==========================
```

This is extremely useful when you are focusing on a single function and want immediate feedback without waiting for the entire suite to run.

### Running Tests by Keyword Expression (`-k`)

Perhaps the most flexible method is using the `-k` flag, which allows you to select tests based on a keyword expression that matches their names.

**To run tests with "discount" in their name**:

```bash
$ pytest -k "discount" -v
========================= test session starts ==========================
...
collected 4 items / 2 deselected / 2 selected

tests/test_pricing.py::test_when_order_total_is_high_should_apply_10_percent_discount PASSED [ 50%]
tests/test_pricing.py::test_when_order_total_is_at_5_percent_boundary_should_apply_discount PASSED [100%]

=================== 2 passed, 2 deselected in ...s ===================
```

Notice how pytest reports that it `deselected` 2 tests and `selected` 2.

You can also use boolean operators like `and`, `or`, and `not`. The expression is case-sensitive.

**To run tests with "stock" but NOT "purchase" in their name**:

```bash
$ pytest -k "stock and not purchase" -v
========================= test session starts ==========================
...
collected 4 items / 3 deselected / 1 selected

tests/test_inventory.py::test_cannot_purchase_out_of_stock_item PASSED [100%]

=================== 1 passed, 3 deselected in ...s ===================
```

Mastering these selection techniques is key to an efficient testing workflow. You can quickly zero in on the tests relevant to your current task, get fast feedback, and iterate rapidly.
