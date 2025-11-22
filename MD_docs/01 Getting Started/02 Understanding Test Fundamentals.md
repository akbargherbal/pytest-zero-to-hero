# Chapter 2: Understanding Test Fundamentals

## Assertions: The Heart of Testing

## Assertions: The Heart of Testing

At its core, every test performs a simple, crucial action: it checks if something is true. This check is called an **assertion**. If the assertion is true, the test passes. If it's false, the test fails. Everything else in a testing framework is built to support this fundamental operation.

A test without an assertion is like a detective who gathers clues but never identifies a suspect. It might run, but it doesn't prove anything.

Pytest uses Python's built-in `assert` statement, making your tests clean, readable, and Pythonic. Let's see it in action with a simple function to test.

```python
# src/wallet.py

class InsufficientAmount(Exception):
    pass

class Wallet:
    def __init__(self, initial_amount=0):
        self.balance = initial_amount

    def spend_cash(self, amount):
        if self.balance < amount:
            raise InsufficientAmount(f"Not enough available to spend {amount}")
        self.balance -= amount

    def add_cash(self, amount):
        self.balance += amount
```

Now, let's write a test for the `add_cash` method using an `assert` statement.

```python
# tests/test_wallet.py

from src.wallet import Wallet

def test_initial_amount():
    wallet = Wallet()
    assert wallet.balance == 0

def test_wallet_add_cash():
    wallet = Wallet(10)
    wallet.add_cash(20)
    assert wallet.balance == 30
```

Here, `assert wallet.balance == 30` is the heart of the test. It declares our expectation: "After adding 20 to a wallet with 10, the balance *must be* 30."

### The Power of Plain Assertions

If you've used other testing frameworks like Python's built-in `unittest`, you might be familiar with methods like `self.assertEqual()`, `self.assertTrue()`, or `self.assertIn()`.

Pytest intentionally avoids these. Why? This is a core part of the pytest philosophy: **keep tests simple and readable**. Using the plain `assert` statement means you don't have to learn dozens of different assertion methods. You just use the same `assert` you might use for debugging or sanity checks in your application code.

### What Happens When an Assertion Fails?

The real power of pytest's assertion system shines when a test fails. Pytest rewrites Python's `assert` statement to provide incredibly detailed and helpful output, a feature called **assertion introspection**.

Let's introduce a bug into our `add_cash` method to see what happens.

```python
# src/wallet.py (with a bug)

class InsufficientAmount(Exception):
    pass

class Wallet:
    def __init__(self, initial_amount=0):
        self.balance = initial_amount

    def spend_cash(self, amount):
        if self.balance < amount:
            raise InsufficientAmount(f"Not enough available to spend {amount}")
        self.balance -= amount

    def add_cash(self, amount):
        # Oops! A bug is introduced here.
        self.balance += amount * 2
```

Now, let's run our test again.

```bash
$ pytest
=========================== test session starts ============================
...
collected 2 items

tests/test_wallet.py .F                                                [100%]

================================= FAILURES =================================
___________________________ test_wallet_add_cash ___________________________

    def test_wallet_add_cash():
        wallet = Wallet(10)
        wallet.add_cash(20)
>       assert wallet.balance == 30
E       assert 50 == 30
E        +  where 50 = &lt;Wallet object at 0x...&gt;.balance

tests/test_wallet.py:12: AssertionError
========================= short test summary info ==========================
FAILED tests/test_wallet.py::test_wallet_add_cash - assert 50 == 30
======================= 1 failed, 1 passed in ...s =======================
```

This failure report is a goldmine of information. Don't just see it as an error; see it as a detailed map to the problem.

1.  `> assert wallet.balance == 30`: Pytest shows you the exact line that failed.
2.  `E assert 50 == 30`: It tells you what the values were at the moment of failure. The `E` stands for "Error line".
3.  `E  +  where 50 = <Wallet object ...>.balance`: This is the magic of introspection. Pytest inspects the expression and tells you that the value `50` came from `wallet.balance`. You don't have to add `print()` statements or use a debugger to see what the value was.

This rich feedback loop is a cornerstone of pytest's design. It helps you diagnose problems faster by giving you all the context you need directly in the test report.

## Test Functions vs. Other Functions

## Test Functions vs. Other Functions

A pytest test function is, for the most part, a regular Python function. It can contain any valid Python code. However, a few key characteristics distinguish it from a non-test function.

| Characteristic      | Test Function (`test_*`)                               | Regular/Helper Function                               |
| ------------------- | ------------------------------------------------------ | ----------------------------------------------------- |
| **Purpose**         | To verify behavior with assertions.                    | To perform a task, calculate a value, or cause an effect. |
| **Naming**          | Must be prefixed with `test_`.                         | Can have any name that doesn't start with `test_`.    |
| **Return Value**    | Almost never returns a value. Its success is its completion. | Often returns a value.                                |
| **Execution**       | Run automatically by the pytest test runner.           | Called explicitly by your code (or a test function).  |

Let's look at an example. Imagine we need to create a wallet with a specific setup for multiple tests. We could write a helper function.

```python
# tests/test_wallet_helpers.py

from src.wallet import Wallet

# This is a HELPER function, not a test.
# Pytest will ignore it because its name doesn't start with `test_`.
def create_wallet_with_transactions(initial_amount, transactions):
    wallet = Wallet(initial_amount)
    for amount in transactions:
        if amount > 0:
            wallet.add_cash(amount)
        else:
            # spending is represented by negative numbers
            wallet.spend_cash(abs(amount))
    return wallet

# This is a TEST function.
# Pytest will discover and run it.
def test_wallet_creation_with_transactions():
    # It calls our helper function.
    my_wallet = create_wallet_with_transactions(50, [10, -20, 5])
    
    # It contains the assertion.
    assert my_wallet.balance == 45

def test_another_scenario():
    # We can reuse the helper.
    my_wallet = create_wallet_with_transactions(100, [-30, -50])
    assert my_wallet.balance == 20
```

In this example:
- `create_wallet_with_transactions` is a standard Python function. It takes arguments and returns a value. Pytest does not run it directly because its name doesn't start with `test_`. It exists only to support our tests and reduce code duplication.
- `test_wallet_creation_with_transactions` is a test function. It has no arguments (for now) and returns nothing. Its job is to set up a scenario (by calling the helper), execute the code under test, and make an assertion. Pytest will find and run this function.

This separation is crucial. Helper functions make your tests cleaner and more maintainable, while test functions provide the entry points for the test runner. The naming convention is the simple, powerful rule that allows pytest to tell them apart.

## Test Discovery: How Pytest Finds Your Tests

## Test Discovery: How Pytest Finds Your Tests

When you type `pytest` in your terminal, it seems like magic. Pytest scans your project, finds all the relevant files and functions, and runs them. But this isn't magic; it's a well-defined process called **test discovery**. Understanding this process is key to structuring your projects effectively.

Pytest follows a simple set of rules by default:

1.  **File Discovery**: It looks for files named `test_*.py` or `*_test.py`.
2.  **Function Discovery**: Within those files, it looks for functions prefixed with `test_`.
3.  **Class Discovery**: It also discovers methods prefixed with `test_` inside classes prefixed with `Test`. (We'll cover class-based testing in Chapter 10).

### Seeing Discovery in Action

To demystify this process, you can ask pytest to show you what it finds without actually running the tests. This is done with the `--collect-only` flag. It's one of the most useful commands for debugging your test setup.

Let's create a project structure with a few files, some of which should be ignored.

**Project Structure:**
```
project/
├── src/
│   └── wallet.py
└── tests/
    ├── test_wallet.py
    ├── wallet_helpers.py
    └── check_spending.py
```

Here's the content of the new files:

```python
# tests/test_wallet.py
# (This file contains our existing tests like test_wallet_add_cash)
from src.wallet import Wallet

def test_wallet_add_cash():
    wallet = Wallet(10)
    wallet.add_cash(20)
    assert wallet.balance == 30
```

```python
# tests/wallet_helpers.py
# This file contains helper functions, but no tests.
# Its name does not start with `test_`.

def setup_test_user():
    return {"name": "test_user", "id": 123}
```

```python
# tests/check_spending.py
# This file's name does NOT start with `test_`.
# Even though the function inside does, the file will be skipped.

from src.wallet import Wallet

def test_spending_is_correct():
    wallet = Wallet(20)
    wallet.spend_cash(10)
    assert wallet.balance == 10
```

Now, let's run the discovery process from the `project/` directory.

```bash
$ pytest --collect-only
=========================== test session starts ============================
...
collected 1 item

&lt;Module tests/test_wallet.py&gt;
  &lt;Function test_wallet_add_cash&gt;

======================== 1 item collected in ...s ========================
```

The output clearly shows what pytest is thinking:
- It found `tests/test_wallet.py` because the filename matches the `test_*.py` pattern.
- Inside that file, it found the function `test_wallet_add_cash`.
- It completely ignored `tests/wallet_helpers.py` because the filename doesn't match.
- It also ignored `tests/check_spending.py` for the same reason, even though it contains a function named `test_spending_is_correct`.

This demonstrates a critical lesson: **both the file and the function must follow the naming convention.** If you ever write a test and find that it's not running, the first thing to check is your naming and the output of `pytest --collect-only`.

## Naming Conventions That Matter

## Naming Conventions That Matter

We've established that the `test_` prefix is a technical requirement for test discovery. But good naming goes far beyond that. A well-named test function is a form of documentation. It describes exactly what behavior is being verified.

Consider these two test names for the same test:
1.  `test_1()`
2.  `test_spend_cash_raises_exception_on_insufficient_funds()`

When you run pytest, the test report becomes a readable list of your application's features and guarantees.

### The Anatomy of a Good Test Name

A great test name often follows a pattern: `test_UNITOFWORK_SCENARIO_EXPECTEDBEHAVIOR`.

-   **`test_`**: The required prefix.
-   **`UNITOFWORK`**: The function, method, or feature you are testing (e.g., `spend_cash`).
-   **`SCENARIO`**: The specific condition you are testing (e.g., `on_insufficient_funds`).
-   **`EXPECTEDBEHAVIOR`**: What you expect to happen (e.g., `raises_exception`).

Let's write a few tests for our `Wallet` class using this convention.

```python
# tests/test_wallet_naming.py

import pytest
from src.wallet import Wallet, InsufficientAmount

def test_initial_balance_is_zero_by_default():
    wallet = Wallet()
    assert wallet.balance == 0

def test_setting_initial_balance():
    wallet = Wallet(100)
    assert wallet.balance == 100

def test_add_cash_increases_balance():
    wallet = Wallet(10)
    wallet.add_cash(90)
    assert wallet.balance == 100

def test_spend_cash_decreases_balance():
    wallet = Wallet(20)
    wallet.spend_cash(10)
    assert wallet.balance == 10

def test_spend_cash_raises_exception_on_insufficient_funds():
    wallet = Wallet(5)
    with pytest.raises(InsufficientAmount):
        wallet.spend_cash(10)
```

Now, let's run these tests with the verbose (`-v`) flag to see how the names appear in the output.

```bash
$ pytest -v tests/test_wallet_naming.py
=========================== test session starts ============================
...
collected 5 items

tests/test_wallet_naming.py::test_initial_balance_is_zero_by_default PASSED [ 20%]
tests/test_wallet_naming.py::test_setting_initial_balance PASSED         [ 40%]
tests/test_wallet_naming.py::test_add_cash_increases_balance PASSED      [ 60%]
tests/test_wallet_naming.py::test_spend_cash_decreases_balance PASSED    [ 80%]
tests/test_wallet_naming.py::test_spend_cash_raises_exception_on_insufficient_funds PASSED [100%]

============================ 5 passed in ...s ============================
```

Look at that output. It's not just a test report; it's a specification document. Anyone can read this list and understand what the `Wallet` class is supposed to do without ever looking at its source code.

Investing a few extra seconds to name your tests descriptively will save you hours of debugging and documentation work in the future.

## Organizing Tests in Your Project

## Organizing Tests in Your Project

As your project grows, the way you organize your test files becomes increasingly important. There are two primary approaches, and pytest supports both seamlessly.

### Approach 1: Tests in a Dedicated `tests/` Directory (Recommended)

This is the most common and scalable approach. You create a top-level `tests` directory, which mirrors the structure of your source code.

```
my_project/
├── pyproject.toml
├── src/
│   ├── __init__.py
│   ├── app/
│   │   ├── __init__.py
│   │   └── models.py
│   └── utils/
│       ├── __init__.py
│       └── helpers.py
└── tests/
    ├── __init__.py
    ├── app/
    │   ├── __init__.py
    │   └── test_models.py
    └── utils/
        ├── __init__.py
        └── test_helpers.py
```

**Pros:**
-   **Clean Separation**: Your application code and test code are completely separate. This makes it easy to package your application for distribution without including the tests.
-   **Scalability**: This structure scales well for large projects with many modules.
-   **Clarity**: It's immediately obvious where to find tests for any given part of the application.

**Cons:**
-   **Import Path Setup**: You need to make sure your `src` directory is on the Python path so that your tests can import the code they are testing (e.g., `from src.app.models import User`). This is typically handled by your project's configuration (`pyproject.toml`) and installing your project in "editable" mode (`pip install -e .`).

### Approach 2: Tests Alongside Application Code

In this approach, test files live inside your application's source directories.

```
my_project/
├── pyproject.toml
└── src/
    ├── __init__.py
    ├── app/
    │   ├── __init__.py
    │   ├── models.py
    │   └── test_models.py
    └── utils/
        ├── __init__.py
        ├── helpers.py
        └── test_helpers.py
```

**Pros:**
-   **Proximity**: Tests are located right next to the code they test, which some developers find convenient.
-   **Simpler Imports**: Imports are typically simpler relative imports (e.g., `from .models import User`).

**Cons:**
-   **Code Bloat**: Your source directories are cluttered with test files.
-   **Packaging Complexity**: You need to configure your packaging tools to exclude `test_*.py` files when you build a distributable version of your library or application.

For this book and for most real-world projects, we will use and recommend the dedicated `tests/` directory. It establishes a professional, maintainable structure from the start. We will cover project setup in detail in Chapter 3.

## Running Specific Tests

## Running Specific Tests

Running your entire test suite is essential for CI/CD and releases, but during development, it's often slow and unnecessary. When you're working on a specific feature or fixing a single bug, you want to run only the relevant tests. Pytest provides powerful and flexible options for doing just that.

Let's assume we have the following test file:

```python
# tests/test_advanced_wallet.py

from src.wallet import Wallet

def test_add_positive_amount():
    wallet = Wallet(10)
    wallet.add_cash(20)
    assert wallet.balance == 30

def test_add_negative_amount():
    # Let's assume this is a feature we want to prevent
    wallet = Wallet(10)
    wallet.add_cash(-5)
    assert wallet.balance == 10 # Balance should not change

def test_spend_positive_amount():
    wallet = Wallet(20)
    wallet.spend_cash(5)
    assert wallet.balance == 15
```

### Running All Tests in a File

To run only the tests in `test_advanced_wallet.py`, simply provide the path to the file.

```bash
$ pytest tests/test_advanced_wallet.py
=========================== test session starts ============================
...
collected 3 items

tests/test_advanced_wallet.py ...                                      [100%]

============================ 3 passed in ...s ============================
```

### Running a Specific Test Function

If you want to run just one test within a file, you can specify its name after the file path, separated by `::`.

```bash
$ pytest tests/test_advanced_wallet.py::test_add_negative_amount
=========================== test session starts ============================
...
collected 1 item

tests/test_advanced_wallet.py .                                        [100%]

============================ 1 passed in ...s ============================
```

This is incredibly useful when you are focused on fixing a single failing test.

### Running Tests by Keyword or Substring

Perhaps the most powerful selection method is the `-k` flag, which runs tests whose names match a given keyword expression.

Let's say we only want to run tests related to "adding" cash.

```bash
$ pytest -k "add"
=========================== test session starts ============================
...
collected 3 items / 1 deselected

tests/test_advanced_wallet.py ..                                       [100%]

====================== 2 passed, 1 deselected in ...s ======================
```

Pytest found all three tests but deselected `test_spend_positive_amount` because its name doesn't contain "add".

The `-k` expression can be more complex. You can use `and`, `or`, and `not`. For example, to run tests that involve adding but are *not* about negative numbers:

```bash
$ pytest -k "add and not negative"
=========================== test session starts ============================
...
collected 3 items / 2 deselected

tests/test_advanced_wallet.py .                                        [100%]

====================== 1 passed, 2 deselected in ...s ======================
```

This selected only `test_add_positive_amount`.

Mastering these selection techniques will dramatically speed up your development workflow. You can iterate quickly on a small subset of tests, ensuring you get fast feedback as you code.
