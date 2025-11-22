# Chapter 7: Assertions and Error Handling

## Beyond Simple Assertions

## Beyond Simple Assertions

At the heart of every test is an assertion. It's the line in the sand, the statement that declares, "This must be true." In Chapter 2, you learned the basics of `assert`, but its power in pytest goes far beyond simple equality checks. Pytest's magic lies in its ability to take a standard Python `assert` statement and provide incredibly detailed feedback when it fails. This is called "assertion introspection."

Let's explore the versatility of the humble `assert` statement.

### Asserting Equality and Inequality

The most common assertion is checking for equality with `==`.

```python
# test_assertions.py

def test_equality():
    assert 1 + 1 == 2

def test_inequality():
    assert 1 + 1 != 3
```

### Asserting Comparisons

You can use any standard comparison operator, and pytest will give you helpful output if it fails.

```python
# test_assertions.py

def test_greater_than():
    assert 5 > 2

def test_less_than_or_equal():
    assert 3 <= 3
    assert 3 <= 4
```

### Asserting Membership

You can check if an item is present in a collection (like a list, tuple, dictionary, or string) using the `in` keyword.

```python
# test_assertions.py

def test_list_membership():
    items = [1, "apple", 3.14]
    assert "apple" in items

def test_dict_key_membership():
    user_data = {"id": 1, "name": "Alice", "email": "alice@example.com"}
    assert "name" in user_data

def test_substring():
    message = "Hello, Pytest!"
    assert "Pytest" in message
```

### Asserting Identity

Sometimes you need to check if two variables refer to the exact same object in memory, not just that they have the same value. For this, you use `is`. This is common when dealing with singletons or specific object instances.

```python
# test_assertions.py

def test_identity():
    a = [1, 2, 3]
    b = a  # b is a reference to the same object as a
    c = [1, 2, 3]  # c has the same value, but is a different object

    assert b is a
    assert c is not a  # They are not the same object
    assert c == a      # But they are equal in value
```

### Asserting Truthiness

You can assert that a value is "truthy" (evaluates to `True` in a boolean context) or "falsy" (evaluates to `False`).

```python
# test_assertions.py

def test_truthiness():
    assert True
    assert 1
    assert "hello"
    assert [1, 2]

def test_falsiness():
    assert not False
    assert not 0
    assert not ""
    assert not []
```

The key takeaway is that you don't need special functions like `assertEqual()` or `assertTrue()` from other frameworks. You can use plain, idiomatic Python expressions, and pytest will do the hard work of figuring out what went wrong when an assertion fails. We'll see exactly how it does that next.

## Assertion Introspection: Reading Failure Messages

## Assertion Introspection: Reading Failure Messages

This is where pytest truly shines and one of the main reasons developers love it. When an `assert` statement fails, pytest doesn't just tell you `AssertionError`. It "introspects" the expression, meaning it looks inside the objects being compared and gives you a detailed, human-readable report of the difference.

A test failure is not an error; it's data. Learning to read this data is one of the most critical skills in testing.

### The Problem: A Failing Test with Complex Data

Imagine we have a function that processes user data and we want to test it. Let's create a test that intentionally fails so we can analyze the output.

```python
# test_user_profile.py

def get_processed_user_data():
    """A function that returns a user profile, but with a bug."""
    return {
        "id": 101,
        "username": "testuser",
        "email": "test.user@example.com",
        "profile": {
            "theme": "dark",
            "notifications": {
                "email": True,
                "sms": False,  # This is the bug, it should be True
            },
        },
        "roles": ["editor", "contributor"],
    }

def test_user_profile_processing():
    expected_data = {
        "id": 101,
        "username": "testuser",
        "email": "test.user@example.com",
        "profile": {
            "theme": "dark",
            "notifications": {
                "email": True,
                "sms": True,  # We expect this to be True
            },
        },
        "roles": ["editor", "contributor"],
    }
    actual_data = get_processed_user_data()
    assert actual_data == expected_data
```

### Reading the Failure Report

Now, let's run this test with `pytest -v` and carefully dissect the output.

```bash
$ pytest -v test_user_profile.py
=========================== test session starts ============================
...
collected 1 item

test_user_profile.py::test_user_profile_processing FAILED          [100%]

================================= FAILURES =================================
______________________ test_user_profile_processing ______________________

    def test_user_profile_processing():
        expected_data = {
            "id": 101,
            "username": "testuser",
            "email": "test.user@example.com",
            "profile": {
                "theme": "dark",
                "notifications": {
                    "email": True,
                    "sms": True,  # We expect this to be True
                },
            },
            "roles": ["editor", "contributor"],
        }
        actual_data = get_processed_user_data()
>       assert actual_data == expected_data
E       assert {'email': 'te... 'testuser'} == {'email': 'te... 'testuser'}
E         Differing items:
E         {'profile': {'notifications': {'sms': False}}} != {'profile': {'notifications': {'sms': True}}}
E         Full diff:
E         - {'email': 'test.user@example.com',
E         -  'id': 101,
E         -  'profile': {'notifications': {'email': True, 'sms': True}, 'theme': 'dark'},
E         ?                                                      ^^^^
E         + {'email': 'test.user@example.com',
E         +  'id': 101,
E         +  'profile': {'notifications': {'email': True, 'sms': False}, 'theme': 'dark'},
E         ?                                                      ^^^^^
E         -  'roles': ['editor', 'contributor'],
E         -  'username': 'testuser'}
E         +  'roles': ['editor', 'contributor'],
E         +  'username': 'testuser'}

test_user_profile.py:28: AssertionError
========================= 1 failed in ...s ===========================
```

Let's break this down piece by piece:

1.  **`> assert actual_data == expected_data`**: The `>` points to the exact line where the assertion failed.
2.  **`E assert {'email': ...} == {'email': ...}`**: The `E` stands for "Error". Pytest shows you the assertion expression again, often abbreviating large data structures for readability.
3.  **`E   Differing items:`**: This is the magic. Pytest intelligently finds the *smallest possible difference* between the two structures. It tells you exactly where the mismatch is, saving you from manually comparing two giant dictionaries.
    ```
    {'profile': {'notifications': {'sms': False}}} != {'profile': {'notifications': {'sms': True}}}
    ```
    It has drilled down through the nested dictionaries to pinpoint the exact key-value pair that differs.
4.  **`E   Full diff:`**: For more context, pytest provides a full `diff` view, similar to what you'd see in version control systems like Git.
    -   Lines starting with `-` are from the left side of the comparison (`actual_data`).
    -   Lines starting with `+` are from the right side (`expected_data`).
    -   Lines starting with `?` highlight the exact characters that differ within a line. Here, `^^^^` points to `True` and `^^^^^` points to `False`.

Without assertion introspection, you would just get `AssertionError`, and you'd have to manually print both dictionaries and stare at them until you found the difference. Pytest's detailed output turns minutes of frustrating debugging into seconds of clear insight.

## Custom Assertion Messages

## Custom Assertion Messages

Python's `assert` statement allows for an optional message that will be displayed if the assertion fails.

The syntax is: `assert <expression>, <message>`

```python
# test_custom_message.py

def test_account_balance():
    account = {"id": "acc_123", "balance": -50}
    # This assertion will fail
    assert account["balance"] >= 0, f"Account {account['id']} has a negative balance!"
```

When we run this test, the custom message is included in the output.

```bash
$ pytest test_custom_message.py
================================= FAILURES =================================
_________________________ test_account_balance _________________________

    def test_account_balance():
        account = {"id": "acc_123", "balance": -50}
        # This assertion will fail
>       assert account["balance"] >= 0, f"Account {account['id']} has a negative balance!"
E       AssertionError: Account acc_123 has a negative balance!
E       assert -50 >= 0

test_custom_message.py:6: AssertionError
========================= 1 failed in ...s ===========================
```

### To Use or Not to Use?

While custom messages can be useful, a common pytest philosophy is to **rely on assertion introspection first**. Pytest's automatic diffing is often more informative than a manually written message.

**When to use custom messages:**

1.  **When the expression isn't self-explanatory:** If the assertion involves complex logic, a message can clarify the *intent* of the test.
2.  **When testing business rules:** A message can describe the business rule that was violated, which is helpful for non-technical stakeholders.
3.  **When looping:** If an assertion is inside a loop, the message can include the loop variable to identify which iteration failed.

**When to avoid them:**

1.  **When comparing simple values:** `assert x == y` provides a great failure message on its own. A custom message like `assert x == y, "x should be equal to y"` is redundant and adds noise.
2.  **When comparing data structures:** Pytest's diffing is almost always superior to any message you could write.

**Guideline:** Write your assertion first. If the failure message from pytest isn't clear enough, then consider adding a custom message to provide more context.

## Testing Exceptions with pytest.raises()

## Testing Exceptions with pytest.raises()

A crucial part of robust software is handling errors gracefully. This means your code should raise exceptions when it receives invalid input or encounters an error state. Your tests must verify that the correct exceptions are raised under the right circumstances.

### The Wrong Way: `try...except`

A beginner might be tempted to write a test like this:

```python
# test_exceptions_wrong.py

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

# This is NOT the recommended way to test exceptions in pytest
def test_divide_by_zero_wrong():
    try:
        divide(10, 0)
    except ValueError:
        # The exception was raised, so the test passes.
        pass
    else:
        # If no exception was raised, the test should fail.
        assert False, "ValueError was not raised"
```

This approach has several problems:
-   It's verbose and boilerplate-heavy.
-   It can accidentally catch the wrong exception if the `try` block is too large.
-   The test will pass if a `ValueError` is raised for the *wrong reason* inside the `try` block.
-   It doesn't provide a clean way to inspect the exception message or attributes.

### The Pytest Way: `pytest.raises`

Pytest provides a clean, expressive, and safe way to assert that a block of code raises an exception: the `pytest.raises` context manager.

```python
# test_exceptions_right.py
import pytest

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

def test_divide_by_zero():
    with pytest.raises(ValueError):
        divide(10, 0)

def test_divide_by_zero_fails_if_no_exception():
    # This test will fail because no exception is raised
    with pytest.raises(ValueError):
        divide(10, 2)
```

This is much cleaner. The `with pytest.raises(ValueError):` block tells pytest: "I expect the code inside this block to raise a `ValueError`. If it does, the test passes. If it raises a different exception or no exception at all, the test fails."

Running the file above produces a clear failure for the second test:

```
_________________ test_divide_by_zero_fails_if_no_exception __________________
...
Failed: DID NOT RAISE <class 'ValueError'>
```

### Inspecting the Exception

Often, you need to check not only that an exception was raised, but also that it has the correct error message. You can capture the exception info using `as excinfo`.

```python
# test_exceptions_right.py
import pytest

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

def test_divide_by_zero_with_message_check():
    with pytest.raises(ValueError) as excinfo:
        divide(10, 0)
    
    # excinfo is an ExceptionInfo object
    # excinfo.value is the actual exception instance
    assert "Cannot divide by zero" in str(excinfo.value)
```

The `excinfo` object is a wrapper around the actual exception. It gives you access to:
-   `excinfo.type`: The type of the exception (e.g., `ValueError`).
-   `excinfo.value`: The exception instance itself.
-   `excinfo.traceback`: The traceback object.

### Using `match` for Cleaner Message Checks

Checking the message with `str(excinfo.value)` works, but pytest provides an even more convenient way: the `match` parameter. It asserts that the exception message matches a regular expression.

```python
# test_exceptions_right.py
import pytest

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

def test_divide_by_zero_with_match():
    # The match parameter accepts a regex pattern.
    # 'zero' will match "Cannot divide by zero"
    with pytest.raises(ValueError, match="divide by zero"):
        divide(10, 0)
```

Using `match` is often preferred because it's more concise and directly states the expectation in the `pytest.raises` call. It makes the test's intent clearer at a glance.

## Testing Warnings with pytest.warns()

## Testing Warnings with pytest.warns()

Warnings are a way for libraries to inform users about deprecated features, potential bugs, or upcoming changes without raising a full exception. Like exceptions, it's important to test that your code emits the correct warnings when it should.

Pytest provides `pytest.warns`, which works almost identically to `pytest.raises`.

### A Function That Emits a Warning

Let's create a function that uses Python's built-in `warnings` module.

```python
# test_warnings.py
import warnings
import pytest

def old_function(x):
    """This is a deprecated function."""
    warnings.warn("old_function() is deprecated, use new_function() instead.", DeprecationWarning)
    return x * 2

def new_function(x):
    """This is the new function."""
    return x * 2
```

### Testing for a Specific Warning

We can use `pytest.warns` as a context manager to assert that code inside the `with` block triggers a specific type of warning.

```python
# test_warnings.py
# ... (previous code) ...

def test_old_function_emits_warning():
    with pytest.warns(DeprecationWarning):
        old_function(5)

def test_new_function_does_not_emit_warning():
    # This will fail if any warning is emitted
    with pytest.warns(None) as record:
        new_function(5)
    
    # Assert that the 'record' list is empty
    assert len(record) == 0
```

If the code inside `pytest.warns(DeprecationWarning)` does not raise that specific warning, the test will fail with a clear message, just like `pytest.raises`.

### Inspecting the Warning Message

Just like `pytest.raises`, you can use the `match` parameter to check the warning's message.

```python
# test_warnings.py
# ... (previous code) ...

def test_old_function_warning_message():
    with pytest.warns(DeprecationWarning, match="is deprecated, use new_function()"):
        old_function(10)
```

You can also capture the warning objects into a list for more detailed inspection. This is useful if a block of code is expected to emit multiple warnings.

```python
# test_warnings.py
# ... (previous code) ...

def function_with_multiple_warnings():
    warnings.warn("First warning", UserWarning)
    warnings.warn("Second warning", RuntimeWarning)

def test_multiple_warnings():
    with pytest.warns(UserWarning) as record:
        function_with_multiple_warnings()

    # The test only passes if at least one UserWarning was caught.
    # The record contains only the captured warnings of the specified type.
    assert len(record) == 1
    assert "First warning" in str(record[0].message)
```

Using `pytest.warns` ensures that your application's communication about deprecations and potential issues remains correct and reliable as your codebase evolves.

## Context Managers for Advanced Assertions

## Context Managers for Advanced Assertions

You've already seen two powerful examples of context managers used for assertions: `pytest.raises` and `pytest.warns`. This pattern—using a `with` statement to assert something about the code block inside it—is incredibly powerful and can be extended for custom, complex assertions.

A context manager is any object in Python that defines `__enter__` and `__exit__` methods, allowing it to be used with the `with` statement. This is perfect for "setup" and "teardown" logic around a piece of code. In testing, we can use the "teardown" part (`__exit__`) to perform assertions about what happened during the "setup" part (`__enter__` and the `with` block).

### Example: Asserting on Log Messages

Imagine you want to test that a specific log message is emitted by a function. You could redirect `stdout` or mock the logging framework, but a custom context manager provides a much cleaner, reusable solution.

Let's build an `assert_logs` context manager.

```python
# test_advanced_assertions.py
import logging
from contextlib import contextmanager

# This is our custom context manager
@contextmanager
def assert_logs(logger, level, expected_message):
    """A context manager to assert that a logger emits a specific message."""
    # Setup: Capture logs
    from io import StringIO
    log_capture_string = StringIO()
    handler = logging.StreamHandler(log_capture_string)
    logger.addHandler(handler)
    
    # Store original level and set the desired one
    original_level = logger.level
    logger.setLevel(level)

    yield  # The code inside the 'with' block runs here

    # Teardown: Perform assertions
    logger.removeHandler(handler)
    logger.setLevel(original_level)
    
    log_contents = log_capture_string.getvalue()
    assert expected_message in log_contents

# --- Code to be tested ---
app_logger = logging.getLogger("my_app")
app_logger.setLevel(logging.INFO)

def process_data(data):
    if not data:
        app_logger.warning("Received empty data payload.")
        return
    app_logger.info(f"Processing data: {data}")
    # ... processing logic ...

# --- The Test ---
import pytest

def test_process_data_logs_warning_for_empty_payload():
    with assert_logs(app_logger, logging.WARNING, "Received empty data payload."):
        process_data(None)

def test_process_data_logs_info():
    with assert_logs(app_logger, logging.INFO, "Processing data: {'id': 1}"):
        process_data({'id': 1})
```

### How It Works

1.  **`@contextmanager`**: This decorator from Python's standard library lets us write a generator-based context manager, which is simpler than writing a full class.
2.  **Setup (`__enter__`)**: Everything before the `yield` is the setup. We create an in-memory text stream (`StringIO`), add a handler to our logger to redirect its output to our stream, and set the logging level.
3.  **`yield`**: This is where control is passed back to the `with` block in the test. The `process_data()` function runs at this point.
4.  **Teardown (`__exit__`)**: Everything after the `yield` is the teardown. We remove our custom handler to stop capturing logs and restore the logger's original level. This cleanup is critical to avoid interfering with other tests.
5.  **Assertion**: Finally, we get the captured log content from our stream and assert that our expected message is present.

This pattern is extremely powerful. You can create context managers to assert on all kinds of behavior:
-   That a database transaction was committed or rolled back.
-   That a file was created or deleted.
-   That a cache was written to or cleared.
-   That a certain number of API calls were made.

While you won't write custom assertion context managers every day, understanding this pattern helps you recognize the power of `pytest.raises` and `pytest.warns` and gives you a tool for creating highly expressive and reusable assertions for complex scenarios.
