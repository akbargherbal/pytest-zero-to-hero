# Chapter 1: Your First Test in 15 Minutes

## Why We Test (The Quick Case)

## Why We Test (The Quick Case)

Before we write a single line of code, let's answer the most important question: why bother? You've probably written code that "just works" without tests. So why add this extra step?

Imagine you've built a complex feature—a shopping cart, a data processing pipeline, a user authentication system. It works perfectly. A month later, you need to add a new feature or fix a bug elsewhere in the code. You make your changes, and everything seems fine. But unknowingly, you've broken the shopping cart. You won't find out until a customer complains, or worse, until you've lost sales.

**Tests are your safety net.**

They are small, automated scripts that verify your code behaves exactly as you expect. They run in seconds and give you immediate feedback.

-   **Confidence to Change:** With a good test suite, you can refactor, upgrade libraries, and add features without fear. If the tests pass, you know you haven't broken existing functionality.
-   **A Living Document:** Tests describe how your code is supposed to be used. A well-written test is better than outdated documentation.
-   **Faster Development:** This sounds counter-intuitive, but it's true. Instead of manually running your application and clicking through screens to check a change, you run a command that does it for you in milliseconds. This feedback loop is incredibly fast.

In this chapter, we'll build that safety net, starting with a single knot. You'll write, run, and understand your first test in the next few minutes.

## Installing Pytest

## Installing Pytest

Pytest is a third-party package, so we'll install it using `pip`, Python's package installer. It's highly recommended to do this inside a **virtual environment** to keep your project dependencies isolated. We'll cover virtual environments in detail in Chapter 3, but for now, let's just get pytest installed.

Open your terminal or command prompt and run the following command:

```bash
pip install pytest
```

You should see output indicating that `pytest` and its dependencies were successfully installed. To verify the installation, you can ask pytest for its version number:

```bash
pytest --version
```

This should print the version of pytest you just installed, something like this (your version number may be different):

```text
pytest 7.4.2
```

That's it. Pytest is now ready to use.

## Writing Your First Test Function

## Writing Your First Test Function

Let's create a simple project to test. We'll start with a single function that we want to verify.

### ### The Code to Test

First, create a new directory for our project. Let's call it `pytest_project`. Inside that directory, create a file named `main.py`.

```text
pytest_project/
└── main.py
```

Now, add a simple function to `main.py`. This is the "production" code we want to test.

```python
# main.py

def add(x, y):
    """Adds two numbers together."""
    return x + y

def subtract(x, y):
    """Subtracts two numbers."""
    return x - y
```

### ### The Test File

Next, we need to create a file to hold our tests. Pytest has a simple but powerful way of discovering tests: it looks for files with names that start or end with `test_`.

Create a new file in the same directory named `test_main.py`.

```python
# test_main.py

# We need to import the code we want to test
from main import add

def test_add():
    """
    Tests the add function with positive integers.
    """
    # The core of a test is a simple 'assert' statement.
    # It checks if a condition is True.
    assert add(2, 3) == 5
    assert add(10, 5) == 15
```

Let's break this down:

1.  **`import from main`**: We import the `add` function from our `main.py` file so we can call it in our test.
2.  **`def test_add():`**: This is our test function. Just like with filenames, pytest automatically discovers any function whose name starts with `test_`. This is the fundamental convention you need to know.
3.  **`assert add(2, 3) == 5`**: This is the heart of the test. An `assert` statement is a built-in Python keyword that checks if a condition is true. If the condition is true, the test continues. If it's false, the test fails immediately and reports an `AssertionError`. Here, we are asserting that the result of calling `add(2, 3)` *is equal to* `5`.

Our project structure now looks like this:

```text
pytest_project/
├── main.py
└── test_main.py
```

## Running Your First Test

## Running Your First Test

Now for the magic moment. Make sure your terminal is in the `pytest_project` directory. Then, simply run the `pytest` command with no arguments.

```bash
pytest
```

Pytest will automatically scan your current directory and subdirectories for any files named `test_*.py` or `*_test.py`, and within those files, it will run any functions prefixed with `test_`.

You should see output similar to this:

```text
============================= test session starts ==============================
platform linux -- Python 3.11.4, pytest-7.4.2, pluggy-1.3.0
rootdir: /path/to/pytest_project
collected 1 item

test_main.py .                                                         [100%]

============================== 1 passed in 0.01s ===============================
```

The important parts are:

-   **`collected 1 item`**: Pytest found one test function (`test_add`).
-   **`test_main.py .`**: The `.` (dot) next to the filename means the test in that file passed.
-   **`1 passed in ...s`**: The final summary confirms that everything was successful.

Congratulations! You've just written and run your first automated test.

## Understanding Test Results

## Understanding Test Results

A passing test is great, but the real value of a testing framework is how it helps you when things go wrong. Let's intentionally make our test fail to see what happens.

### ### A Failing Test

Modify `test_main.py` to check for an incorrect result.

```python
# test_main.py

from main import add

def test_add():
    """
    Tests the add function with positive integers.
    """
    assert add(2, 3) == 5
    # Let's add a failing assertion
    assert add(10, 5) == 10  # This is wrong! 10 + 5 is 15, not 10.
```

Now, run `pytest` again from your terminal.

```bash
pytest
```

This time, the output is very different. It's a detailed failure report, and learning to read it is one of the most important skills in testing.

```text
============================= test session starts ==============================
platform linux -- Python 3.11.4, pytest-7.4.2, pluggy-1.3.0
rootdir: /path/to/pytest_project
collected 1 item

test_main.py F                                                         [100%]

=================================== FAILURES ===================================
___________________________________ test_add ___________________________________

    def test_add():
        """
        Tests the add function with positive integers.
        """
        assert add(2, 3) == 5
        # Let's add a failing assertion
>       assert add(10, 5) == 10  # This is wrong! 10 + 5 is 15, not 10.
E       assert 15 == 10
E        +  where 15 = add(10, 5)

test_main.py:9: AssertionError
=========================== short test summary info ============================
FAILED test_main.py::test_add - assert 15 == 10
============================== 1 failed in 0.03s ===============================
```

### ### Dissecting the Failure Report

This report is data, not just an error. It's a map telling you exactly where the problem is.

1.  **`test_main.py F`**: The `.` has been replaced by an `F`, giving you an immediate visual cue that a test failed.
2.  **`FAILURES` section**: This is the detailed breakdown.
3.  **`_________________ test_add _________________`**: It tells you the exact test function that failed.
4.  **The Code Traceback**: It shows you the lines of code leading up to the failure. The line marked with `>` is the exact line that failed: `assert add(10, 5) == 10`.
5.  **Assertion Introspection (The Magic)**: This is the most powerful part. Pytest doesn't just say "assertion failed." It inspects the assertion and shows you the *actual values* involved.
    -   `E assert 15 == 10`: It shows the comparison it tried to make.
    -   `E  +  where 15 = add(10, 5)`: It goes even further, showing you that the left side of the comparison, `add(10, 5)`, evaluated to `15`.

You don't have to guess or add `print()` statements to debug. Pytest tells you, "I expected `10`, but the function `add(10, 5)` actually returned `15`."

Now, go back to `test_main.py`, fix the assertion, and run `pytest` one more time to see it pass.

```python
# test_main.py (corrected)

from main import add

def test_add():
    """
    Tests the add function with positive integers.
    """
    assert add(2, 3) == 5
    assert add(10, 5) == 15 # Corrected assertion
```

Running `pytest` now will bring you back to the satisfying `1 passed` message.

## What You've Accomplished

## What You've Accomplished

In just a few minutes, you have performed the fundamental workflow of testing that professionals use every day:

1.  **You wrote application code** (`main.py`).
2.  **You wrote a test** (`test_main.py`) that defines the correct behavior of that code.
3.  **You ran the test suite** using the `pytest` command and saw it pass, giving you confidence that your code works.
4.  **You saw a test failure** and learned how to read the detailed report to instantly diagnose the problem.

This simple `write code -> write test -> run test` loop is the foundation upon which all modern software development is built. You've created a safety net for the `add` function. Now, no matter what other changes you make to your project, you can always run `pytest` to ensure this core piece of logic still works as intended.

In the next chapter, we will build on these fundamentals, exploring assertions in more detail and learning more about how pytest's powerful test discovery really works.
