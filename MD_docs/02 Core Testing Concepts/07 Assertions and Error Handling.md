# Chapter 7: Assertions and Error Handling

## Beyond Simple Assertions

## The Heart of the Test: The `assert` Statement

At its core, every test performs three steps: Arrange, Act, and Assert. The assertion is the moment of truth—it's where we verify that the outcome of our action matches our expectation. Pytest is built around Python's native `assert` statement, supercharging it with a feature called "assertion introspection." This means you don't need special assertion methods like `self.assertEqual()` or `self.assertTrue()` that are common in other frameworks. A plain `assert` is all you need.

This chapter is dedicated to mastering assertions. We'll start with the basics, learn how to read pytest's incredibly detailed failure reports, and then move on to advanced techniques for handling expected errors and warnings.

### Phase 1: Establish the Reference Implementation

To explore these concepts, we need a realistic piece of code to test. We'll build a simple `UserValidator` that checks if a user data dictionary is valid for registration. This will be our **anchor example** for the entire chapter.

Our validator will check three things:
1.  The user must be at least 18 years old.
2.  The username must not be "admin".
3.  The email must contain an "@" symbol.

Here is the initial implementation of our validator and its first test.

```python
# user_validator.py

from datetime import date

class UserValidator:
    def __init__(self, user_data: dict):
        self.data = user_data
        self.errors = {}

    def _validate_age(self):
        """User must be 18 or older."""
        today = date.today()
        try:
            birthdate = date.fromisoformat(self.data.get("birthdate", ""))
            age = today.year - birthdate.year - ((today.month, today.day) < (birthdate.month, birthdate.day))
            if age < 18:
                self.errors["age"] = "User must be 18 or older"
        except (ValueError, TypeError):
            self.errors["birthdate"] = "Invalid date format"

    def _validate_username(self):
        """Username cannot be 'admin'."""
        username = self.data.get("username")
        if username == "admin":
            self.errors["username"] = "Username cannot be 'admin'"

    def _validate_email(self):
        """Email must be valid."""
        email = self.data.get("email", "")
        if "@" not in email:
            self.errors["email"] = "Invalid email address"

    def is_valid(self) -> bool:
        """Runs all validations and returns True if no errors."""
        self.errors = {} # Reset errors on each run
        self._validate_age()
        self._validate_username()
        self._validate_email()
        return not self.errors

    def get_validation_report(self) -> dict:
        """Runs validation and returns a detailed report."""
        self.is_valid() # Ensure validation has run
        return {
            "is_valid": not self.errors,
            "errors": self.errors
        }

# test_validator.py

from user_validator import UserValidator

def test_valid_user():
    """Tests a user with completely valid data."""
    valid_data = {
        "username": "john_doe",
        "birthdate": "2001-05-15",
        "email": "john.doe@example.com"
    }
    validator = UserValidator(valid_data)
    
    # We expect the validation report to show success
    expected_report = {
        "is_valid": True,
        "errors": {}
    }
    
    actual_report = validator.get_validation_report()
    
    assert actual_report == expected_report
```

Let's run this to confirm our baseline is working.

```bash
$ pytest -v
========================= test session starts ==========================
...
collected 1 item

test_validator.py::test_valid_user PASSED                         [100%]

========================== 1 passed in ...s ==========================
```

Great. Now, let's write a test for an invalid user who is underage. This will expose our first assertion failure and set the stage for learning how to read pytest's output.

```python
# test_validator.py (added test)

from user_validator import UserValidator
from datetime import date, timedelta

def test_valid_user():
    # ... (same as before) ...
    pass

def test_underage_user():
    """Tests a user who is younger than 18."""
    # A birthdate exactly 17 years ago
    underage_birthdate = (date.today() - timedelta(days=17*365)).isoformat()
    
    invalid_data = {
        "username": "jane_doe",
        "birthdate": underage_birthdate,
        "email": "jane.doe@example.com"
    }
    validator = UserValidator(invalid_data)
    
    expected_report = {
        "is_valid": False,
        "errors": {
            "age": "User must be 18 or older"
        }
    }
    
    actual_report = validator.get_validation_report()
    
    assert actual_report == expected_report
```

This test seems correct. We create a user who is clearly underage and assert that the validation report reflects this specific error. Let's run it.

```bash
$ pytest -v
========================= test session starts ==========================
...
collected 2 items

test_validator.py::test_valid_user PASSED                         [ 50%]
test_validator.py::test_underage_user FAILED                      [100%]

=============================== FAILURES ===============================
__________________________ test_underage_user __________________________

    def test_underage_user():
        """Tests a user who is younger than 18."""
        # A birthdate exactly 17 years ago
        underage_birthdate = (date.today() - timedelta(days=17*365)).isoformat()
    
        invalid_data = {
            "username": "jane_doe",
            "birthdate": underage_birthdate,
            "email": "jane.doe@example.com"
        }
        validator = UserValidator(invalid_data)
    
        expected_report = {
            "is_valid": False,
            "errors": {
                "age": "User must be 18 or older"
            }
        }
    
        actual_report = validator.get_validation_report()
    
>       assert actual_report == expected_report
E       AssertionError: assert {'errors': {'...id': False} == {'errors': {'...id': False}
E         Differing items:
E         {'errors': {'age': 'User must be 18 or older'}} != {'errors': {'age': 'User must be at least 18'}}
E         Full diff:
E         - {'errors': {'age': 'User must be at least 18'}, 'is_valid': False}
E         ?                                  ---------
E         + {'errors': {'age': 'User must be 18 or older'}, 'is_valid': False}
E         ?                                  ^^^^^^^^

test_validator.py:36: AssertionError
======================= short test summary info ========================
FAILED test_validator.py::test_underage_user - AssertionError: assert...
===================== 1 failed, 1 passed in ...s =====================
```

It failed! This is perfect. The failure isn't due to a complex logical bug, but a simple typo in our test's expectation. This provides an ideal opportunity to learn how to dissect pytest's rich failure reports.

## Assertion Introspection: Reading Failure Messages

## Diagnostic Analysis: Reading the Failure

A failing test is not a dead end; it's a map that tells you exactly where your code deviates from your expectations. Learning to read this map is one of the most critical skills in testing. Pytest's output is dense with information, so let's break down the failure from the previous section piece by piece.

**The complete output**:
```bash
=============================== FAILURES ===============================
__________________________ test_underage_user __________________________

    def test_underage_user():
        """Tests a user who is younger than 18."""
        # ... (code omitted for brevity) ...
    
        actual_report = validator.get_validation_report()
    
>       assert actual_report == expected_report
E       AssertionError: assert {'errors': {'...id': False} == {'errors': {'...id': False}
E         Differing items:
E         {'errors': {'age': 'User must be 18 or older'}} != {'errors': {'age': 'User must be at least 18'}}
E         Full diff:
E         - {'errors': {'age': 'User must be at least 18'}, 'is_valid': False}
E         ?                                  ---------
E         + {'errors': {'age': 'User must be 18 or older'}, 'is_valid': False}
E         ?                                  ^^^^^^^^

test_validator.py:36: AssertionError
```

**Let's parse this section by section**:

1.  **The summary line**: `FAILED test_validator.py::test_underage_user - AssertionError: assert...`
    -   **What this tells us**: The test named `test_underage_user` in the file `test_validator.py` failed. The reason for the failure was an `AssertionError`, which means the condition in an `assert` statement evaluated to `False`.

2.  **The traceback**:
    ```python
    >       assert actual_report == expected_report
    test_validator.py:36: AssertionError
    ```
    -   **What this tells us**: This points to the exact line of code that failed. The `>` character highlights line 36 in `test_validator.py`. This is the most important piece of information for locating the problem.

3.  **The assertion introspection**:
    ```
    E       AssertionError: assert {'errors': {'...id': False} == {'errors': {'...id': False}
    E         Differing items:
    E         {'errors': {'age': 'User must be 18 or older'}} != {'errors': {'age': 'User must be at least 18'}}
    E         Full diff:
    E         - {'errors': {'age': 'User must be at least 18'}, 'is_valid': False}
    E         ?                                  ---------
    E         + {'errors': {'age': 'User must be 18 or older'}, 'is_valid': False}
    E         ?                                  ^^^^^^^^
    ```
    -   **What this tells us**: This is pytest's magic. It "introspects" the objects involved in the failed assertion and provides a detailed, human-readable diff.
        -   The first line shows the assertion that failed, with large data structures abbreviated (`...`).
        -   `Differing items:` gets to the heart of the matter. It isolates the exact key-value pair in the dictionary that didn't match. We can clearly see the string for the `'age'` error is different.
        -   `Full diff:` provides a standard `diff` format. Lines starting with `-` are from the left side of the `==` (the expected value in our case), and lines with `+` are from the right side (the actual value). The `?` lines highlight the precise characters that differ.

**Root cause identified**: We have a typo in our test. The `UserValidator` produces the error message `"User must be 18 or older"`, but our test *expected* `"User must be at least 18"`.

**Why the current approach can't solve this**: The approach is fine, but our data is wrong. The introspection gave us all the information we needed to spot the mistake instantly.

**What we need**: We need to fix the expected string in our test to match the actual implementation.

Let's apply the fix.

```python
# test_validator.py (fixed)

# ... (imports and test_valid_user are the same) ...

def test_underage_user():
    """Tests a user who is younger than 18."""
    underage_birthdate = (date.today() - timedelta(days=17*365)).isoformat()
    
    invalid_data = {
        "username": "jane_doe",
        "birthdate": underage_birthdate,
        "email": "jane.doe@example.com"
    }
    validator = UserValidator(invalid_data)
    
    # BEFORE: The expected message was wrong
    # expected_report = {
    #     "is_valid": False,
    #     "errors": {
    #         "age": "User must be at least 18"
    #     }
    # }

    # AFTER: The expected message now matches the implementation
    expected_report = {
        "is_valid": False,
        "errors": {
            "age": "User must be 18 or older"
        }
    }
    
    actual_report = validator.get_validation_report()
    
    assert actual_report == expected_report
```

Now, running pytest shows both tests passing.

```bash
$ pytest -v
========================= test session starts ==========================
...
collected 2 items

test_validator.py::test_valid_user PASSED                         [ 50%]
test_validator.py::test_underage_user PASSED                      [100%]

========================== 2 passed in ...s ==========================
```

This simple exercise demonstrates the power of assertion introspection. Without it, we would have just seen `AssertionError` and would have had to manually print both dictionaries to find the difference. Pytest saves you that step, accelerating the debug cycle.

## Custom Assertion Messages

## Iteration 1: Adding Context with Custom Messages

Pytest's introspection is fantastic for comparing two objects, but sometimes the failure isn't about a difference between `a` and `b`. Sometimes, the failure is that a single value isn't what you expect it to be, and the reason *why* you expect it isn't obvious from the code.

### Current Limitation

Our tests are good at checking the final `report` dictionary. But what if we wanted to test a property of the validator itself? Let's add a feature to count the number of validation rules.

```python
# user_validator.py (updated)

class UserValidator:
    def __init__(self, user_data: dict):
        self.data = user_data
        self.errors = {}
        # A list of all validation methods
        self._validations = [
            self._validate_age,
            self._validate_username,
            self._validate_email
        ]

    @property
    def validation_rule_count(self):
        return len(self._validations)

    # ... (_validate methods are the same) ...

    def is_valid(self) -> bool:
        """Runs all validations and returns True if no errors."""
        self.errors = {}
        for validation_func in self._validations:
            validation_func()
        return not self.errors
    
    # ... (get_validation_report is the same) ...
```

Now, let's write a test to ensure we always have exactly three validation rules. This acts as a safeguard against someone accidentally removing a rule.

```python
# test_validator.py (added test)

# ... (other tests) ...

def test_validation_rule_count():
    """Ensures the number of validation rules is correct."""
    validator = UserValidator({}) # Data doesn't matter for this test
    assert validator.validation_rule_count == 3
```

This test passes. But now, let's simulate a future change where a developer *removes* a validation rule, causing our safeguard test to fail.

```python
# user_validator.py (temporarily broken)

class UserValidator:
    def __init__(self, user_data: dict):
        self.data = user_data
        self.errors = {}
        # A developer accidentally removed a validation!
        self._validations = [
            self._validate_age,
            self._validate_username,
            # self._validate_email is missing!
        ]
    # ... (rest of the class) ...
```

### Failure Demonstration

Let's run pytest now.

```bash
$ pytest -v
========================= test session starts ==========================
...
collected 3 items

test_validator.py::test_valid_user PASSED                         [ 33%]
test_validator.py::test_underage_user PASSED                      [ 66%]
test_validator.py::test_validation_rule_count FAILED              [100%]

=============================== FAILURES ===============================
______________________ test_validation_rule_count ______________________

    def test_validation_rule_count():
        """Ensures the number of validation rules is correct."""
        validator = UserValidator({}) # Data doesn't matter for this test
>       assert validator.validation_rule_count == 3
E       assert 2 == 3
E        +  where 2 = <user_validator.UserValidator object at 0x...>.validation_rule_count

test_validator.py:42: AssertionError
======================= short test summary info ========================
FAILED test_validator.py::test_validation_rule_count - assert 2 == 3
===================== 1 failed, 2 passed in ...s =====================
```

### Diagnostic Analysis: Reading the Failure

**The complete output**: (Shown above)

**Let's parse this section by section**:

1.  **The summary line**: `FAILED test_validator.py::test_validation_rule_count - assert 2 == 3`
    -   **What this tells us**: The test failed because an assertion expected `3` but got `2`.

2.  **The traceback**:
    ```python
    >       assert validator.validation_rule_count == 3
    test_validator.py:42: AssertionError
    ```
    -   **What this tells us**: The failure is on line 42.

3.  **The assertion introspection**:
    ```
    E       assert 2 == 3
    E        +  where 2 = <user_validator.UserValidator object at 0x...>.validation_rule_count
    ```
    -   **What this tells us**: Pytest does its best. It shows that the value `2` came from the property `validation_rule_count`. This is helpful, but it doesn't explain the *business logic* behind the number `3`. Why was `3` the magic number?

**Root cause identified**: The number of validation rules has changed from 3 to 2.
**Why the current approach is limited**: The failure message is technically correct but lacks context. A developer seeing `assert 2 == 3` might not immediately understand the significance.
**What we need**: A way to add a human-readable message to the failure output to explain the *intent* of the assertion.

### Technique: Custom Assertion Messages

Python's `assert` statement has a second, optional argument: a message to display if the assertion fails.

```python
assert <condition>, "This is the message that will be shown on failure."
```

Let's add a descriptive message to our test.

```python
# test_validator.py (improved test)

def test_validation_rule_count():
    """Ensures the number of validation rules is correct."""
    validator = UserValidator({})
    
    # BEFORE
    # assert validator.validation_rule_count == 3

    # AFTER
    message = (
        "The number of validation rules has changed. "
        "If this was intentional, update the test. "
        "Rules should cover: age, username, email."
    )
    assert validator.validation_rule_count == 3, message
```

### Verification

Now, let's run the test again with our broken `UserValidator` (that still has only 2 rules).

```bash
$ pytest -v
========================= test session starts ==========================
...
collected 3 items

test_validator.py::test_valid_user PASSED                         [ 33%]
test_validator.py::test_underage_user PASSED                      [ 66%]
test_validator.py::test_validation_rule_count FAILED              [100%]

=============================== FAILURES ===============================
______________________ test_validation_rule_count ______________________

    def test_validation_rule_count():
        """Ensures the number of validation rules is correct."""
        validator = UserValidator({})
    
        message = (
            "The number of validation rules has changed. "
            "If this was intentional, update the test. "
            "Rules should cover: age, username, email."
        )
>       assert validator.validation_rule_count == 3, message
E       AssertionError: The number of validation rules has changed. If this was intentional, update the test. Rules should cover: age, username, email.
E       assert 2 == 3
E        +  where 2 = <user_validator.UserValidator object at 0x...>.validation_rule_count

test_validator.py:50: AssertionError
======================= short test summary info ========================
FAILED test_validator.py::test_validation_rule_count - AssertionError: The...
===================== 1 failed, 2 passed in ...s =====================
```

Look at that! The custom message is now the first thing you see in the failure report. It immediately tells the developer what the number `3` means and what they should check. This transforms a simple `assert 2 == 3` into a rich, actionable piece of feedback.

(Remember to fix the `UserValidator` class by re-adding `_validate_email` to the list before proceeding.)

## Testing Exceptions with pytest.raises()

## Iteration 2: Handling Expected Errors

Our current `UserValidator` is polite. When it finds invalid data, it adds an error to a dictionary. This is a valid design pattern, but often, invalid input should be treated more severely by raising an exception. This signals a programming error—the caller should have sanitized the data *before* passing it to the object.

Let's refactor our validator. Instead of collecting errors, the `_validate_age` method will now raise a `ValueError` if the birthdate format is wrong.

### Current Limitation

Our tests are designed to check the state of the `errors` dictionary. They have no mechanism to handle or expect exceptions.

### New Scenario: Invalid Date Format

Let's change `_validate_age` to be stricter.

```python
# user_validator.py (refactored)

from datetime import date

class UserValidator:
    # ... (__init__ and other _validate methods are the same for now) ...
    
    def _validate_age(self):
        """User must be 18 or older. Raises ValueError on bad date format."""
        today = date.today()
        birthdate_str = self.data.get("birthdate")
        if not birthdate_str:
            self.errors["birthdate"] = "Birthdate is required"
            return
            
        try:
            birthdate = date.fromisoformat(birthdate_str)
            age = today.year - birthdate.year - ((today.month, today.day) < (birthdate.month, birthdate.day))
            if age < 18:
                self.errors["age"] = "User must be 18 or older"
        except ValueError:
            # This is the new behavior!
            raise ValueError(f"Invalid date format for birthdate: '{birthdate_str}'")

    # ... (rest of the class) ...
```

How do we test this? A naive approach might be to write a test that calls the validator with a bad date and see what happens.

```python
# test_validator.py (added test)

# ... (other tests) ...

def test_invalid_date_format_raises_exception():
    """
    Tests that a malformed birthdate string raises a ValueError.
    """
    invalid_data = {
        "username": "test_user",
        "birthdate": "2000-13-01", # Invalid month
        "email": "test@example.com"
    }
    validator = UserValidator(invalid_data)
    
    # How do we assert that an exception is raised?
    # This line will crash the test.
    validator.is_valid()
```

### Failure Demonstration

Running this new test results in an `ERROR`, not a `FAIL`. This is a crucial distinction. A `FAIL` means an assertion was proven false. An `ERROR` means the test itself crashed due to an unhandled exception.

```bash
$ pytest -v
========================= test session starts ==========================
...
collected 4 items

test_validator.py::test_valid_user PASSED                         [ 25%]
test_validator.py::test_underage_user PASSED                      [ 50%]
test_validator.py::test_validation_rule_count PASSED              [ 75%]
test_validator.py::test_invalid_date_format_raises_exception ERROR [100%]

================================ ERRORS ================================
ERROR at setup of test_invalid_date_format_raises_exception
...
    def test_invalid_date_format_raises_exception():
        # ...
        validator = UserValidator(invalid_data)
    
        # This line will crash the test.
>       validator.is_valid()

test_validator.py:59: 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
user_validator.py:22: in is_valid
    validation_func()
user_validator.py:36: in _validate_age
    raise ValueError(f"Invalid date format for birthdate: '{birthdate_str}'")
E   ValueError: Invalid date format for birthdate: '2000-13-01'
======================= short test summary info ========================
ERROR test_validator.py::test_invalid_date_format_raises_exception
===================== 1 error, 3 passed in ...s ======================
```

### Diagnostic Analysis: Reading the Failure

**The complete output**: (Shown above)

1.  **The summary line**: `ERROR test_validator.py::test_invalid_date_format_raises_exception`
    -   **What this tells us**: The test didn't fail an assertion; it crashed. This is an uncontrolled error.

2.  **The traceback**:
    ```
    user_validator.py:36: in _validate_age
        raise ValueError(f"Invalid date format for birthdate: '{birthdate_str}'")
    E   ValueError: Invalid date format for birthdate: '2000-13-01'
    ```
    -   **What this tells us**: The traceback clearly shows a `ValueError` was raised from within our `_validate_age` method, exactly as we designed. The problem is that our test didn't catch it.

**Root cause identified**: The test code does not handle the `ValueError` that the application code correctly raises.
**Why the current approach can't solve this**: A test function, by default, is expected to run to completion without raising exceptions. An unhandled exception is always an error.
**What we need**: A way to tell pytest, "I expect this specific block of code to raise this specific exception. If it does, the test passes. If it doesn't, or if it raises a different exception, the test fails."

### Technique: The `pytest.raises` Context Manager

Pytest provides a clean, expressive context manager for this exact purpose: `pytest.raises`.

You use it like this:
```python
import pytest

def test_something_that_raises():
    with pytest.raises(ExpectedException):
        # Code that is expected to raise ExpectedException
        call_my_function()
```

If `call_my_function()` raises `ExpectedException`, the `with` block catches it, and the test passes. If it raises a different exception or no exception at all, the test fails.

Let's fix our test using this pattern.

```python
# test_validator.py (fixed with pytest.raises)
import pytest
# ... (other imports and tests) ...

def test_invalid_date_format_raises_exception():
    """
    Tests that a malformed birthdate string raises a ValueError.
    """
    invalid_data = {
        "username": "test_user",
        "birthdate": "2000-13-01", # Invalid month
        "email": "test@example.com"
    }
    validator = UserValidator(invalid_data)
    
    # BEFORE: This crashed the test
    # validator.is_valid()

    # AFTER: We wrap the call in pytest.raises
    with pytest.raises(ValueError):
        validator.is_valid()
```

### Verification

Running pytest now shows all tests passing.

```bash
$ pytest -v
========================= test session starts ==========================
...
collected 4 items

test_validator.py::test_valid_user PASSED                         [ 25%]
test_validator.py::test_underage_user PASSED                      [ 50%]
test_validator.py::test_validation_rule_count PASSED              [ 75%]
test_validator.py::test_invalid_date_format_raises_exception PASSED [100%]

========================== 4 passed in ...s ==========================
```

### Asserting on Exception Details

Passing is good, but we can be more specific. What if the function raises a `ValueError` for the wrong reason? The `pytest.raises` context manager can give you access to the exception object itself.

You can capture it using `as excinfo`:

```python
# test_validator.py (enhanced test)

def test_invalid_date_format_raises_exception_with_message():
    """
    Tests that a malformed birthdate string raises a ValueError
    with a specific error message.
    """
    bad_date = "2000-13-01"
    invalid_data = {
        "username": "test_user",
        "birthdate": bad_date,
        "email": "test@example.com"
    }
    validator = UserValidator(invalid_data)
    
    with pytest.raises(ValueError) as excinfo:
        validator.is_valid()
    
    # excinfo is an ExceptionInfo object
    # .value is the actual exception instance
    assert bad_date in str(excinfo.value)
    assert "Invalid date format" in str(excinfo.value)

    # A more convenient way is to use the match parameter
    match_str = f"Invalid date format for birthdate: '{bad_date}'"
    with pytest.raises(ValueError, match=match_str):
        validator.is_valid()
```

The `match` parameter is particularly useful. It checks the exception's string representation against a regular expression. This makes your test more robust and readable.

### Common Failure Modes and Their Signatures

#### Symptom: The expected exception was never raised.

**Pytest output pattern**:
```
>       with pytest.raises(ValueError):
E       Failed: DID NOT RAISE <class 'ValueError'>

test_validator.py:65: Failed
```
**Diagnostic clues**: The output explicitly says `DID NOT RAISE`.
**Root cause**: The code inside the `with` block completed without raising the specified exception. This indicates a bug in your application code—it's not failing when it should.
**Solution**: Debug the application code to find out why the exception isn't being raised under the test conditions.

#### Symptom: A different exception was raised.

**Pytest output pattern**:
```
>       with pytest.raises(ValueError):
>           raise TypeError("Something else went wrong")
E       TypeError: Something else went wrong

...
During handling of the above exception, another exception occurred:
...
E       Failed: DID NOT RAISE <class 'ValueError'>
```
**Diagnostic clues**: You'll see the traceback for the *unexpected* exception (`TypeError` in this case), followed by the `Failed: DID NOT RAISE <class 'ValueError'>` message.
**Root cause**: The code raised an exception, but not the one you were expecting. This could be a bug in your application or a mistake in your test's expectation.
**Solution**: Analyze the unexpected exception. Is it the correct behavior? If so, update your test to expect `TypeError`. If not, fix the application code to raise `ValueError` as intended.

## Testing Warnings with pytest.warns()

## Iteration 3: Capturing Expected Warnings

Exceptions are for errors, but what about conditions that aren't errors yet but might be in the future? Python has a built-in warning system for this, commonly used for deprecation notices.

Let's evolve our `UserValidator`. Suppose we decide that the `username` field is being deprecated in favor of using the `email` as the primary identifier. We want to keep the username validation for now, but we want to warn the user whenever it's being validated.

### Current Limitation

Our tests check for return values and exceptions, but they are completely blind to warnings. A test would pass, and the warning would be printed to the console, but it wouldn't be programmatically verified.

### New Scenario: Deprecated Username Validation

Let's modify `_validate_username` to issue a `DeprecationWarning`.

```python
# user_validator.py (updated)
import warnings
# ...

class UserValidator:
    # ...
    def _validate_username(self):
        """Username cannot be 'admin'. Issues a warning."""
        warnings.warn(
            "'username' is a deprecated field.",
            DeprecationWarning
        )
        username = self.data.get("username")
        if username == "admin":
            self.errors["username"] = "Username cannot be 'admin'"
    # ...
```

If we run our existing tests, they will pass, but we'll see a `DeprecationWarning` in the output. This is okay, but it's not a guarantee. We want a test that *fails* if this warning is *not* issued.

### Failure Demonstration

How can we make a test fail if a warning isn't present? A common CI/CD practice is to treat warnings as errors to keep the codebase clean. We can simulate this using the `-W error` pytest flag.

```bash
$ pytest -v -W error
========================= test session starts ==========================
...
collected 5 items

test_validator.py::test_valid_user FAILED                         [ 20%]
test_validator.py::test_underage_user FAILED                      [ 40%]
test_validator.py::test_validation_rule_count PASSED              [ 60%]
test_validator.py::test_invalid_date_format_raises_exception_with_message PASSED [ 80%]
test_validator.py::test_invalid_date_format_raises_exception PASSED [100%]

=============================== FAILURES ===============================
____________________________ test_valid_user ___________________________
...
>       validator.is_valid()
E       DeprecationWarning: 'username' is a deprecated field.
...
======================= short test summary info ========================
FAILED test_validator.py::test_valid_user - DeprecationWarning: 'user...
FAILED test_validator.py::test_underage_user - DeprecationWarning: 'u...
===================== 2 failed, 3 passed in ...s =====================
```

### Diagnostic Analysis: Reading the Failure

**The complete output**: (Shown above)

1.  **The summary line**: `FAILED test_validator.py::test_valid_user - DeprecationWarning: ...`
    -   **What this tells us**: The test failed because it encountered a `DeprecationWarning`, which we told pytest to treat as an error.

2.  **The traceback**:
    ```
    >       validator.is_valid()
    E       DeprecationWarning: 'username' is a deprecated field.
    ```
    -   **What this tells us**: The warning was triggered by the call to `validator.is_valid()`.

**Root cause identified**: Our code is correctly issuing a warning, but our tests aren't explicitly handling it. When running in a strict mode (`-W error`), this unhandled warning becomes a failure.
**Why the current approach can't solve this**: We have no mechanism to declare "this warning is expected and should be ignored for this specific test."
**What we need**: A tool similar to `pytest.raises` but for warnings.

### Technique: The `pytest.warns` Context Manager

Unsurprisingly, pytest provides `pytest.warns`, which has an almost identical API to `pytest.raises`.

```python
import pytest

def test_something_that_warns():
    with pytest.warns(ExpectedWarning, match="..."):
        # Code that is expected to issue ExpectedWarning
        call_my_function()
```

This block will catch the expected warning, allowing the test to pass (even with `-W error`), and fail if the warning is not issued.

Let's write a new test specifically for this warning.

```python
# test_validator.py (added test)
import pytest
# ...

def test_username_validation_issues_deprecation_warning():
    """
    Tests that validating a user with a username issues a DeprecationWarning.
    """
    data = {
        "username": "test_user",
        "birthdate": "2000-01-01",
        "email": "test@example.com"
    }
    validator = UserValidator(data)

    with pytest.warns(DeprecationWarning, match="'username' is a deprecated field."):
        validator.is_valid()
```

### Verification

Now, when we run pytest, this new test will pass, and importantly, we can update our other tests to ignore this specific warning if needed, though it's often better to have a dedicated test for the warning and ensure other tests use data that doesn't trigger it.

```bash
$ pytest -v
========================= test session starts ==========================
...
collected 6 items

...
test_validator.py::test_username_validation_issues_deprecation_warning PASSED [100%]

=================== 5 passed, 1 skipped in ...s ====================
(Note: Previous tests might show warnings in the output if not run with -W error, but the new test passes cleanly)
```

### Common Failure Modes and Their Signatures

#### Symptom: The expected warning was not issued.

**Pytest output pattern**:
```
>       with pytest.warns(DeprecationWarning):
E       Failed: DID NOT WARN.

test_my_code.py:10: Failed
```
**Diagnostic clues**: The output is a very clear `DID NOT WARN`.
**Root cause**: The code inside the `with` block completed without issuing any warnings of the specified type.
**Solution**: Debug the application code to see why the `warnings.warn()` call is not being reached.

## Context Managers for Advanced Assertions

## Beyond Built-ins: Custom Assertion Contexts

We've seen how `pytest.raises` and `pytest.warns` are powerful tools. They work by wrapping a piece of code in a context manager (`with ...:`) and asserting something about the *behavior* of that code block—did it raise? did it warn?

This pattern is not limited to pytest's built-in tools. You can create your own assertion context managers to verify complex behaviors that are difficult to check with a simple `assert` on a return value. This is an advanced technique, but it demonstrates the power and extensibility of Python's testing patterns.

### Scenario: Asserting on Side Effects

Let's imagine our `UserValidator` has a new requirement: when a user is successfully validated, their email must be added to a "welcome list" in a database.

```python
# user_validator.py (final version)
# ... (imports)

class MockDB:
    """A fake database for demonstration purposes."""
    def __init__(self):
        self.welcome_list = []
    
    def add_to_welcome_list(self, email):
        print(f"Adding {email} to welcome list.")
        self.welcome_list.append(email)

class UserValidator:
    def __init__(self, user_data: dict, db: MockDB):
        self.data = user_data
        self.db = db
        # ... (rest of __init__)

    # ... (other methods) ...

    def is_valid(self) -> bool:
        """
        Runs all validations. If successful, adds user email to the
        welcome list.
        """
        self.errors = {}
        for validation_func in self._validations:
            validation_func()
        
        is_successful = not self.errors
        if is_successful:
            self.db.add_to_welcome_list(self.data.get("email"))
            
        return is_successful
```

How do we test this? We could do this:

```python
# test_validator.py (a simple test for the side effect)

from user_validator import MockDB

def test_valid_user_is_added_to_welcome_list():
    db = MockDB()
    valid_data = {
        "username": "john_doe",
        "birthdate": "2001-05-15",
        "email": "john.doe@example.com"
    }
    validator = UserValidator(valid_data, db)
    
    validator.is_valid()
    
    assert "john.doe@example.com" in db.welcome_list
```

This works, but what if we want to create a more general and expressive assertion? For example, "I assert that executing this block of code adds exactly one user to the welcome list."

### Technique: Creating a Custom Assertion Context Manager

We can build our own context manager that checks the state of the database *before* and *after* the code block runs.

```python
# test_validator.py (with a custom context manager)

import pytest
from contextlib import contextmanager
from user_validator import UserValidator, MockDB

@contextmanager
def assert_adds_to_welcome_list(db: MockDB, expected_count: int):
    """
    A context manager to assert that a block of code adds a specific
    number of users to the welcome list.
    """
    # ARRANGE: Get the state *before* the action
    initial_count = len(db.welcome_list)
    
    # ACT: Yield to let the code inside the 'with' block run
    yield
    
    # ASSERT: Get the state *after* and compare
    final_count = len(db.welcome_list)
    added_count = final_count - initial_count
    
    assert added_count == expected_count, (
        f"Expected to add {expected_count} user(s) to welcome list, "
        f"but added {added_count}."
    )

# New test using the context manager
def test_valid_user_is_added_to_welcome_list_with_context_manager():
    db = MockDB()
    valid_data = {
        "username": "john_doe",
        "birthdate": "2001-05-15",
        "email": "john.doe@example.com"
    }
    validator = UserValidator(valid_data, db)
    
    with assert_adds_to_welcome_list(db, expected_count=1):
        validator.is_valid()

# A test for the negative case
def test_invalid_user_is_not_added_to_welcome_list():
    db = MockDB()
    invalid_data = {"username": "admin"} # This will fail validation
    validator = UserValidator(invalid_data, db)

    with assert_adds_to_welcome_list(db, expected_count=0):
        validator.is_valid()
```

This custom context manager makes the test's intent incredibly clear. It reads like a sentence: "Assert that running `validator.is_valid()` adds exactly one user to the welcome list."

This pattern is extremely powerful for testing code with side effects, such as database interactions, file system changes, or API calls. It encapsulates the setup and teardown logic for the assertion, keeping the test itself clean and focused on the action. It's the same principle that makes `pytest.raises` and `pytest.warns` so effective, applied to your own application's logic.

### The Journey: From Problem to Solution

| Iteration | Failure Mode                               | Technique Applied          | Result                                                              |
| --------- | ------------------------------------------ | -------------------------- | ------------------------------------------------------------------- |
| 0         | Simple assertion on a dictionary failed    | Assertion Introspection    | Pytest's diff pinpointed the exact string mismatch in our test data.  |
| 1         | `assert 2 == 3` lacked context             | Custom Assertion Message   | The failure report now explains the business logic behind the numbers.  |
| 2         | An expected exception crashed the test     | `pytest.raises()`          | The test now correctly expects and catches the `ValueError`.          |
| 3         | An expected warning was unverified         | `pytest.warns()`           | The test now guarantees that a `DeprecationWarning` is issued.        |
| 4         | Asserting on a side-effect was verbose     | Custom Context Manager     | The test's intent is now declarative and encapsulated.                |

### Decision Framework: Which Assertion Approach When?

| Assertion Type                 | Use Case                                                              | Example                                                              |
| ------------------------------ | --------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `assert a == b`                | Verifying the state or return value of an object. The most common case. | `assert user.is_active() is True`                                    |
| `assert a == b, "message"`     | The `assert` is simple, but the *reason* for the expected value is not. | `assert len(items) == 5, "API should return 5 items per page"`       |
| `with pytest.raises(Error):`   | Verifying that a specific block of code *must* raise an exception.      | `with pytest.raises(KeyError): my_dict["missing"]`                   |
| `with pytest.warns(Warning):`  | Verifying that a specific block of code *must* issue a warning.         | `with pytest.warns(FutureWarning): legacy_function()`                 |
| Custom Context Manager         | Verifying a complex side effect (e.g., DB change, file write, API call).| `with assert_db_commit(): service.save()`                            |

### Lessons Learned

-   **Treat every failure as data.** Pytest's output is a rich report, not a simple "pass/fail." Learn to read the traceback, introspection diffs, and summary lines to diagnose problems instantly.
-   **Use the right tool for the job.** Don't write `try/except` blocks in your tests when `pytest.raises` exists. Use the framework's idiomatic tools to make your tests clearer and more concise.
-   **Assert on intent.** A good assertion verifies not just *what* happened, but that it happened for the *right reason*. Adding custom messages or using `pytest.raises` with `match` makes your tests more precise.
-   **Think beyond return values.** Tests should verify behavior and side effects, not just the output of a function. Context managers are the primary pattern for asserting on behavior over time.
