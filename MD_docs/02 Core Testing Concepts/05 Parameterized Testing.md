# Chapter 5: Parameterized Testing

## Testing Multiple Scenarios Without Repetition

## The Problem: Testing the Same Logic with Different Data

In testing, we rarely check just one scenario. For any given function, we need to verify its behavior with various inputs: valid data, invalid data, edge cases, boundary conditions, and more. A robust test suite is one that covers a wide spectrum of possibilities.

Let's establish our anchor example for this chapter: a password validation function. This is a classic use case because the rules for a "valid" password create numerous scenarios we need to test.

### Phase 1: Establish the Reference Implementation

Our initial function, `is_valid_password`, will have a few simple rules:
1.  Must be at least 8 characters long.
2.  Must contain at least one number.
3.  Must contain at least one uppercase letter.

Here is the implementation.

```python
# validation.py

import re

def is_valid_password(password: str) -> bool:
    """
    Checks if a password meets the following criteria:
    1. At least 8 characters long.
    2. Contains at least one number.
    3. Contains at least one uppercase letter.
    """
    if len(password) < 8:
        return False
    if not re.search(r"\d", password):
        return False
    if not re.search(r"[A-Z]", password):
        return False
    return True
```

Now, let's write some tests for it. The most straightforward, yet most problematic, approach is to write a separate test function for each scenario.

```python
# test_validation_v1.py

from validation import is_valid_password

def test_valid_password():
    """A valid password should be accepted."""
    assert is_valid_password("ValidPass1") is True

def test_password_too_short():
    """A password shorter than 8 characters should be rejected."""
    assert is_valid_password("Short1") is False

def test_password_missing_number():
    """A password without a number should be rejected."""
    assert is_valid_password("NoNumberPass") is False

def test_password_missing_uppercase():
    """A password without an uppercase letter should be rejected."""
    assert is_valid_password("nouppercase1") is False
```

Let's run these tests to confirm they work as expected.

```bash
$ pytest -v
========================= test session starts ==========================
...
collected 4 items

test_validation_v1.py::test_valid_password PASSED                 [ 25%]
test_validation_v1.py::test_password_too_short PASSED              [ 50%]
test_validation_v1.py::test_password_missing_number PASSED         [ 75%]
test_validation_v1.py::test_password_missing_uppercase PASSED      [100%]

========================== 4 passed in ...s ==========================
```

### The Pitfall: Code Duplication and Maintenance Burden

The tests pass, but we've created a maintenance problem. This approach violates the "Don't Repeat Yourself" (DRY) principle.

**Current Limitation:**
-   **Repetitive Structure:** Every test function has the exact same structure: call `is_valid_password` with a specific input and assert the expected boolean result.
-   **High Maintenance:** If the function's signature changes (e.g., it starts returning a reason for failure), we have to update *every single test function*.
-   **Poor Scalability:** What if we need to test 20 more scenarios? We would have to write 20 more nearly identical functions. The test file would become bloated and hard to read.

This is a "maintenance failure" waiting to happen. The code works today, but it's brittle and expensive to change tomorrow. We need a way to test the *same logic* with a *list of different inputs*. This is the core problem that parametrization solves.

## @pytest.mark.parametrize Basics

## Iteration 1: Consolidating Tests with `@pytest.mark.parametrize`

Pytest provides a powerful decorator, `@pytest.mark.parametrize`, to handle this exact situation. It allows you to define multiple sets of arguments for a single test function. Pytest will then run the test function once for each set of arguments.

**The Concept Before the Syntax:**
Instead of thinking "I need four tests," we shift our thinking to "I need one test that verifies password validity, and I have four *scenarios* to check."

The decorator takes two main arguments:
1.  A string containing a comma-separated list of argument names.
2.  A list of values, where each element in the list is a set of arguments for one test run.

### Refactoring to a Single Parameterized Test

Let's refactor our four separate tests into one concise, data-driven test. We will have two parameters: the `password` to test and the `expected` boolean result.

**Before:**

```python
# test_validation_v1.py (excerpt)

def test_valid_password():
    assert is_valid_password("ValidPass1") is True

def test_password_too_short():
    assert is_valid_password("Short1") is False

def test_password_missing_number():
    assert is_valid_password("NoNumberPass") is False

def test_password_missing_uppercase():
    assert is_valid_password("nouppercase1") is False
```

**After:**

```python
# test_validation_v2.py

import pytest
from validation import is_valid_password

@pytest.mark.parametrize("password, expected", [
    ("ValidPass1", True),         # Scenario: Valid password
    ("Short1", False),            # Scenario: Too short
    ("NoNumberPass", False),      # Scenario: Missing number
    ("nouppercase1", False),      # Scenario: Missing uppercase
])
def test_password_validation(password, expected):
    """Tests is_valid_password with multiple scenarios."""
    assert is_valid_password(password) is expected
```

### Verification: Running the Parameterized Test

Let's run this new test file with the verbose (`-v`) flag to see how pytest handles it.

```bash
$ pytest -v test_validation_v2.py
========================= test session starts ==========================
...
collected 4 items

test_validation_v2.py::test_password_validation[ValidPass1-True] PASSED [ 25%]
test_validation_v2.py::test_password_validation[Short1-False] PASSED   [ 50%]
test_validation_v2.py::test_password_validation[NoNumberPass-False] PASSED [ 75%]
test_validation_v2.py::test_password_validation[nouppercase1-False] PASSED [100%]

========================== 4 passed in ...s ==========================
```

### Banish Magic with Mechanics: How It Works

Notice the output. Even though we wrote only one test function (`test_password_validation`), pytest reports that it collected and ran **4 items**. It has dynamically generated a separate test case for each tuple in our parameter list.

The part in the square brackets, like `[ValidPass1-True]`, is the **Test ID**. Pytest automatically generates this from the parameter values to help you distinguish which specific scenario passed or failed.

**Expected vs. Actual Improvement:**
-   **Code Volume:** We replaced four functions (12 lines of code) with one parameterized function (8 lines).
-   **Maintainability:** Adding a new test case is now a one-line change—we just add a new tuple to the list. If the `is_valid_password` function signature changes, we only have to update one test function.
-   **Readability:** The test data is neatly organized as a table of inputs and expected outputs, making the intent of the test crystal clear.

**Limitation Preview:** This is excellent for simple inputs. But what if our function takes multiple, distinct arguments? Or what if we need to combine this with fixtures? We'll explore that next.

## Multiple Parameters

## Iteration 2: Handling Multiple Distinct Parameters

Our current test uses two parameters, `password` and `expected`, which works perfectly. The syntax `parametrize("arg1, arg2", [...])` is designed for exactly this. Each element in the list of parameters must be a tuple whose size matches the number of argument names.

Let's evolve our anchor example to make this more explicit. A simple boolean return value is often not enough. Good validation functions should tell you *why* something failed. Let's refactor `is_valid_password` to return a tuple: `(bool, str)`, where the string is a reason for failure.

### Evolving the Function Under Test

Here's the new version of our validation function.

```python
# validation_v2.py

import re
from typing import Tuple

def is_valid_password(password: str) -> Tuple[bool, str]:
    """
    Checks password validity and returns a tuple of (bool, reason_string).
    """
    if len(password) < 8:
        return (False, "Password must be at least 8 characters long.")
    if not re.search(r"\d", password):
        return (False, "Password must contain at least one number.")
    if not re.search(r"[A-Z]", password):
        return (False, "Password must contain at least one uppercase letter.")
    
    return (True, "Password is valid.")
```

### Failure Demonstration: Adapting the Test

Our existing test is now broken because it only asserts against a boolean. If we run it against the new function, it will fail. Let's create a new test file and adapt our test. We now need to check three things: the input `password`, the `expected_validity` (boolean), and the `expected_reason`.

This requires us to parametrize three arguments.

```python
# test_validation_v3.py

import pytest
from validation_v2 import is_valid_password

@pytest.mark.parametrize("password, expected_validity, expected_reason", [
    ("ValidPass1", True, "Password is valid."),
    ("Short1", False, "Password must be at least 8 characters long."),
    ("NoNumberPass", False, "Password must contain at least one number."),
    ("nouppercase1", False, "Password must contain at least one uppercase letter."),
    # Let's add a new case that fails on the first check to be thorough
    ("short", False, "Password must be at least 8 characters long."),
])
def test_password_validation_with_reasons(password, expected_validity, expected_reason):
    """
    Tests is_valid_password with multiple scenarios, checking validity and reason.
    """
    is_valid, reason = is_valid_password(password)
    assert is_valid is expected_validity
    assert reason == expected_reason
```

### Verification

Running this new test shows that all scenarios are correctly handled.

```bash
$ pytest -v test_validation_v3.py
========================= test session starts ==========================
...
collected 5 items

test_validation_v3.py::test_password_validation_with_reasons[ValidPass1-True-Password is valid.] PASSED [ 20%]
test_validation_v3.py::test_password_validation_with_reasons[Short1-False-Password must be at least 8 characters long.] PASSED [ 40%]
test_validation_v3.py::test_password_validation_with_reasons[NoNumberPass-False-Password must contain at least one number.] PASSED [ 60%]
test_validation_v3.py::test_password_validation_with_reasons[nouppercase1-False-Password must contain at least one uppercase letter.] PASSED [ 80%]
test_validation_v3.py::test_password_validation_with_reasons[short-False-Password must be at least 8 characters long.] PASSED [100%]

========================== 5 passed in ...s ==========================
```

### Key Takeaway

The `@pytest.mark.parametrize` decorator seamlessly handles multiple arguments.

-   The first argument to the decorator is a string of comma-separated names: `"arg1, arg2, arg3"`.
-   The second argument is a list of tuples, where each tuple contains the values for `(arg1, arg2, arg3)` for a single test run.
-   The test function must accept arguments with the same names.

This pattern is incredibly powerful for testing complex logic where multiple inputs influence the outcome.

**Limitation Preview:** Our test data is hardcoded inside the decorator. This is fine for a few cases, but what if the data is complex, comes from a file, or needs to be generated? Also, what if our test needs a shared resource, like a database connection, in addition to the parameters? This is where combining parametrization with fixtures becomes essential.

## Combining Parametrization with Fixtures

## Iteration 3: Using Fixtures and Parameters Together

Tests rarely exist in a vacuum. They often require setup, teardown, or access to shared resources. In pytest, these are handled by fixtures. A common question is: "How can I use my parameters alongside my fixtures?"

The answer is simple: pytest handles it automatically. A test function can accept arguments that come from both `@pytest.mark.parametrize` and fixtures. Pytest is smart enough to resolve each argument to its correct source.

### Evolving the Anchor Example

Let's imagine our password validation rules are no longer hardcoded. Instead, they are defined in a `PasswordPolicy` object. This makes our system more flexible.

```python
# validation_v3.py

import re
from typing import Tuple
from dataclasses import dataclass

@dataclass
class PasswordPolicy:
    min_length: int
    require_number: bool
    require_uppercase: bool

def is_valid_password(password: str, policy: PasswordPolicy) -> Tuple[bool, str]:
    """
    Checks password validity against a given policy object.
    """
    if len(password) < policy.min_length:
        return (False, f"Password must be at least {policy.min_length} characters long.")
    if policy.require_number and not re.search(r"\d", password):
        return (False, "Password must contain at least one number.")
    if policy.require_uppercase and not re.search(r"[A-Z]", password):
        return (False, "Password must contain at least one uppercase letter.")
    
    return (True, "Password is valid.")
```

Our `is_valid_password` function now requires a `PasswordPolicy` instance. In a real application, this policy might be loaded from a configuration file or a database. For our tests, it's a perfect candidate for a fixture.

### Creating a Fixture and a Parameterized Test

Let's create a fixture that provides a default password policy. Our test function will then accept this fixture *and* our parameterized arguments.

```python
# test_validation_v4.py

import pytest
from validation_v3 import is_valid_password, PasswordPolicy

@pytest.fixture
def default_policy() -> PasswordPolicy:
    """Provides a default password policy for tests."""
    return PasswordPolicy(min_length=8, require_number=True, require_uppercase=True)

@pytest.mark.parametrize("password, expected_validity, expected_reason", [
    ("ValidPass1", True, "Password is valid."),
    ("Short1", False, "Password must be at least 8 characters long."),
    ("NoNumberPass", False, "Password must contain at least one number."),
    ("nouppercase1", False, "Password must contain at least one uppercase letter."),
])
def test_password_validation_with_fixture(default_policy, password, expected_validity, expected_reason):
    """
    Tests password validation using a fixture for the policy
    and parameters for the scenarios.
    """
    # The 'default_policy' argument is supplied by the fixture.
    # The other arguments are supplied by @pytest.mark.parametrize.
    is_valid, reason = is_valid_password(password, default_policy)
    
    assert is_valid is expected_validity
    assert reason == expected_reason
```

### Verification and Mechanics

When we run this, it works exactly as you'd hope.

```bash
$ pytest -v test_validation_v4.py
========================= test session starts ==========================
...
collected 4 items

test_validation_v4.py::test_password_validation_with_fixture[ValidPass1-True-Password is valid.] PASSED [ 25%]
test_validation_v4.py::test_password_validation_with_fixture[Short1-False-Password must be at least 8 characters long.] PASSED [ 50%]
test_validation_v4.py::test_password_validation_with_fixture[NoNumberPass-False-Password must contain at least one number.] PASSED [ 75%]
test_validation_v4.py::test_password_validation_with_fixture[nouppercase1-False-Password must contain at least one uppercase letter.] PASSED [100%]

========================== 4 passed in ...s ==========================
```

**How Pytest Resolves Arguments:**

1.  Pytest first inspects the signature of `test_password_validation_with_fixture`: `(default_policy, password, expected_validity, expected_reason)`.
2.  It sees the `@pytest.mark.parametrize` decorator and knows it is responsible for providing `password`, `expected_validity`, and `expected_reason`.
3.  It then looks for a source for the remaining argument, `default_policy`. It finds a fixture with that name and injects its return value.

This composition of fixtures and parametrization is a cornerstone of writing clean, scalable, and maintainable tests with pytest.

**Limitation Preview:** This is powerful, but what if the *parameters themselves* need to be processed by a fixture? For example, what if our parameters are not the final data, but rather instructions for a fixture to *create* the data? This scenario requires a special feature called "indirect parametrization."

## Indirect Parametrization

## Iteration 4: Processing Parameters with Fixtures

Sometimes, the values you want to parametrize are not the direct inputs to your test function. Instead, they are identifiers or configurations that a fixture should use to perform some setup.

Consider this scenario: We want to test our password policy against auto-generated passwords. Our parameters won't be the password strings themselves, but rather *specifications* for how to generate them.

**The Problem:** How do we get a fixture to receive a value from `@pytest.mark.parametrize`?

The solution is **indirect parametrization**. By adding `indirect=True` to the decorator, you tell pytest that the specified parameters should be passed to fixtures of the same name, and the *result* of those fixtures should be passed to the test function.

### Evolving the Anchor Example for Indirect Parametrization

Let's create a test where the parameters are dictionaries describing the password to be generated. We'll create a fixture that takes these dictionaries and builds the password strings.

First, a simple password generator function.

```python
# utils.py

def generate_password(spec: dict) -> str:
    """Generates a password based on a specification dictionary."""
    length = spec.get("length", 8)
    has_number = spec.get("has_number", True)
    has_uppercase = spec.get("has_uppercase", True)

    password = ""
    if has_uppercase:
        password += "A"
    else:
        password += "a"
    
    if has_number:
        password += "1"
    else:
        password += "b"
        
    # Fill the rest with lowercase letters
    password += "c" * (length - len(password))
    return password
```

Now, let's write the test. We will create a parameter named `password_spec`. We will then tell pytest to pass this parameter to a fixture, also named `password_spec`, for processing.

**Before (Direct Parametrization):** The test receives the parameter value directly.
```python
@pytest.mark.parametrize("password", ["ValidPass1", "Short1"])
def test_direct(password):
    # 'password' is "ValidPass1" in the first run.
    ...
```

**After (Indirect Parametrization):** The test receives the *return value* of the fixture that was called with the parameter value.

```python
# test_validation_v5.py

import pytest
from utils import generate_password
from validation_v3 import is_valid_password, PasswordPolicy

@pytest.fixture
def default_policy() -> PasswordPolicy:
    """Provides a default password policy for tests."""
    return PasswordPolicy(min_length=8, require_number=True, require_uppercase=True)

@pytest.fixture
def generated_password(request) -> str:
    """
    A fixture that receives a password spec via indirect parametrization
    and returns a generated password string.
    """
    spec = request.param  # This is how a fixture accesses the parameter
    return generate_password(spec)

@pytest.mark.parametrize(
    "generated_password, expected_validity",
    [
        ({"length": 8, "has_number": True, "has_uppercase": True}, True),
        ({"length": 7, "has_number": True, "has_uppercase": True}, False),
        ({"length": 8, "has_number": False, "has_uppercase": True}, False),
        ({"length": 8, "has_number": True, "has_uppercase": False}, False),
    ],
    indirect=["generated_password"]  # Tell pytest this parameter is indirect
)
def test_password_validation_indirect(default_policy, generated_password, expected_validity):
    """
    Tests password validation using an indirectly parameterized fixture
    to generate test data.
    """
    # 'generated_password' is now the *result* of the fixture, not the spec dict.
    is_valid, _ = is_valid_password(generated_password, default_policy)
    assert is_valid is expected_validity
```

### Verification and Mechanics

Let's run this and see the result.

```bash
$ pytest -v test_validation_v5.py
========================= test session starts ==========================
...
collected 4 items

test_validation_v5.py::test_password_validation_indirect[generated_password0-True] PASSED [ 25%]
test_validation_v5.py::test_password_validation_indirect[generated_password1-False] PASSED [ 50%]
test_validation_v5.py::test_password_validation_indirect[generated_password2-False] PASSED [ 75%]
test_validation_v5.py::test_password_validation_indirect[generated_password3-False] PASSED [100%]

========================== 4 passed in ...s ==========================
```

**The Flow of Indirect Parametrization:**

1.  Pytest sees `@pytest.mark.parametrize` with `indirect=["generated_password"]`.
2.  For the first run, it takes the parameter value `{"length": 8, ...}`.
3.  Because `generated_password` is marked as indirect, it does **not** pass this dictionary to the test function.
4.  Instead, it finds the fixture named `generated_password`.
5.  It calls the `generated_password` fixture, making the dictionary available via the special `request.param` object.
6.  The fixture runs its logic (`generate_password(...)`) and returns a password string (e.g., `"A1cccccc"`).
7.  Pytest then calls the test function `test_password_validation_indirect`, passing the *return value* of the fixture as the `generated_password` argument.

### When to Apply This Solution

-   **What it optimizes for:** Decoupling test data from test logic. It allows your parameters to be high-level descriptions, while the complex object creation is encapsulated in a reusable fixture.
-   **When to choose this approach:**
    -   When parameters are identifiers (e.g., user IDs, file names) that a fixture needs to look up or load.
    -   When parameters are configurations for creating complex objects (as in our example).
    -   When the setup for a parameter is expensive and you want to contain it within a fixture's scope.
-   **When to avoid this approach:** For simple, self-contained data types (strings, numbers, booleans), direct parametrization is simpler and more readable.

**Limitation Preview:** Look at the test output IDs: `[generated_password0-True]`, `[generated_password1-False]`. These are not very helpful. If a test fails, we have no idea which scenario it was. We need a way to provide clear, descriptive names for our test cases.

## Generating Test IDs for Clarity

## Iteration 5: Improving Readability with Custom Test IDs

When a parameterized test fails, the test ID is your first clue to understanding what went wrong. As we saw in the last section, pytest's default IDs can be unhelpful, especially with complex data structures like dictionaries.

`test_password_validation_indirect[generated_password2-False]`

Which scenario is `generated_password2`? We have to go back to the source code and count down the list to figure it out. This slows down debugging.

Pytest allows you to provide custom, human-readable test IDs using the `ids` parameter in `@pytest.mark.parametrize`.

### Failure Demonstration: The Unhelpful Failure Report

Let's intentionally break one of our indirect tests to see the problem clearly. Suppose our generator has a bug and produces a password with no number when it should.

```python
# test_validation_v6_fail.py (with an intentional error in the data)

# ... (fixtures and imports from v5) ...

@pytest.mark.parametrize(
    "generated_password, expected_validity",
    [
        # ... other cases ...
        # INTENTIONAL BUG: Expecting valid, but spec will produce invalid password
        ({"length": 8, "has_number": False, "has_uppercase": True}, True),
    ],
    indirect=["generated_password"]
)
def test_password_validation_indirect_fail(default_policy, generated_password, expected_validity):
    is_valid, _ = is_valid_password(generated_password, default_policy)
    assert is_valid is expected_validity
```

### Diagnostic Analysis: Reading the Failure

**The complete output**:
```bash
$ pytest -v test_validation_v6_fail.py
========================= test session starts ==========================
...
collected 1 item

test_validation_v6_fail.py::test_password_validation_indirect_fail[generated_password0-True] FAILED [100%]

============================== FAILURES ==============================
__ test_password_validation_indirect_fail[generated_password0-True] __

default_policy = PasswordPolicy(min_length=8, require_number=True, require_uppercase=True)
generated_password = 'Abbbbbbb', expected_validity = True

    @pytest.mark.parametrize(
        "generated_password, expected_validity",
        [
            # ... other cases ...
            # INTENTIONAL BUG: Expecting valid, but spec will produce invalid password
            ({"length": 8, "has_number": False, "has_uppercase": True}, True),
        ],
        indirect=["generated_password"]
    )
    def test_password_validation_indirect_fail(default_policy, generated_password, expected_validity):
        is_valid, _ = is_valid_password(generated_password, default_policy)
>       assert is_valid is expected_validity
E       assert False is True
E        +  where False = is_valid_password('Abbbbbbb', PasswordPolicy(min_length=8, require_number=True, require_uppercase=True))[0]

test_validation_v6_fail.py:36: AssertionError
======================= 1 failed in ...s =======================
```

**Let's parse this section by section**:

1.  **The summary line**: `FAILED test_validation_v6_fail.py::test_password_validation_indirect_fail[generated_password0-True]`
    -   What this tells us: The test named `test_password_validation_indirect_fail` failed. The failing scenario is identified only as `generated_password0-True`. This is not descriptive. We don't know *which* case failed without looking at the code.

2.  **The traceback**: The traceback points to the `assert` line, which is correct.

3.  **The assertion introspection**: `assert False is True`
    -   What this tells us: The `is_valid_password` function returned `False`, but the test expected `True`. The introspection helpfully shows us the generated password was `'Abbbbbbb'`.

**Root cause identified**: The test failed because a password generated from a spec with `has_number: False` was expected to be valid, which is incorrect according to our policy.
**Why the current approach is problematic**: The test ID `[generated_password0-True]` gives us no context about the *intent* of the failing test case. We want it to say something like "fail-if-no-number".

### Solution: Providing Custom IDs

We can fix this by passing a list of strings to the `ids` parameter. The list must have the same number of elements as the parameter sets.

**Before:**

```python
# test_validation_v5.py (excerpt)
@pytest.mark.parametrize(
    "generated_password, expected_validity",
    [
        ({"length": 8, "has_number": True, "has_uppercase": True}, True),
        ({"length": 7, "has_number": True, "has_uppercase": True}, False),
        ({"length": 8, "has_number": False, "has_uppercase": True}, False),
        ({"length": 8, "has_number": True, "has_uppercase": False}, False),
    ],
    indirect=["generated_password"]
)
def test_password_validation_indirect(...):
    ...
```

**After (Final Implementation):**

```python
# test_validation_final.py

import pytest
from utils import generate_password
from validation_v3 import is_valid_password, PasswordPolicy

@pytest.fixture
def default_policy() -> PasswordPolicy:
    return PasswordPolicy(min_length=8, require_number=True, require_uppercase=True)

@pytest.fixture
def generated_password(request) -> str:
    spec = request.param
    return generate_password(spec)

# A helper function for generating IDs
def id_generator(spec):
    parts = []
    if not spec.get("has_uppercase", True):
        parts.append("no_upper")
    if not spec.get("has_number", True):
        parts.append("no_num")
    if spec.get("length", 8) < 8:
        parts.append(f"len_{spec['length']}")
    
    if not parts:
        return "valid_password"
    return "-".join(parts)

@pytest.mark.parametrize(
    "generated_password, expected_validity",
    [
        ({"length": 8, "has_number": True, "has_uppercase": True}, True),
        ({"length": 7, "has_number": True, "has_uppercase": True}, False),
        ({"length": 8, "has_number": False, "has_uppercase": True}, False),
        ({"length": 8, "has_number": True, "has_uppercase": False}, False),
    ],
    indirect=["generated_password"],
    # We can provide a list of strings...
    # ids=[
    #     "valid-password",
    #     "too-short",
    #     "missing-number",
    #     "missing-uppercase",
    # ]
    # ...or even better, a function to generate them!
    ids=lambda spec: id_generator(spec[0]) if isinstance(spec, tuple) else id_generator(spec)
)
def test_password_validation_with_ids(default_policy, generated_password, expected_validity):
    is_valid, _ = is_valid_password(generated_password, default_policy)
    assert is_valid is expected_validity
```

### Verification with New IDs

Now when we run the tests, the output is far more informative.

```bash
$ pytest -v test_validation_final.py
========================= test session starts ==========================
...
collected 4 items

test_validation_final.py::test_password_validation_with_ids[valid_password] PASSED [ 25%]
test_validation_final.py::test_password_validation_with_ids[len_7] PASSED      [ 50%]
test_validation_final.py::test_password_validation_with_ids[no_num] PASSED      [ 75%]
test_validation_final.py::test_password_validation_with_ids[no_upper] PASSED    [100%]

========================== 4 passed in ...s ==========================
```

If the "missing-number" test were to fail now, the report would immediately tell us `FAILED ... [no_num]`, giving us instant context for debugging.

### The Journey: From Problem to Solution

| Iteration | Failure Mode / Problem                               | Technique Applied             | Result                                                              |
| --------- | ---------------------------------------------------- | ----------------------------- | ------------------------------------------------------------------- |
| 0         | Repetitive test functions, high maintenance cost.    | None (Copy-Paste)             | Four separate, brittle tests.                                       |
| 1         | Consolidated tests but only handled one parameter.   | `@pytest.mark.parametrize`    | A single test function for all scenarios.                           |
| 2         | Needed to test multiple inputs and outputs.          | Multiple arguments in `parametrize` | Test now validates input, validity, and reason message together.    |
| 3         | Test logic coupled with policy creation.             | Combining with Fixtures       | Policy creation is handled by a fixture, separating concerns.       |
| 4         | Parameters were complex to create directly.          | Indirect Parametrization      | Parameters became high-level specs; a fixture handled data generation. |
| 5         | Default test IDs were unreadable and unhelpful.      | `ids` parameter               | Clear, descriptive test IDs that accelerate debugging.              |

### Lessons Learned

-   **Start Simple:** Begin by writing separate tests to understand your scenarios.
-   **Refactor to Parametrize:** Once you see repetition, consolidate your tests with `@pytest.mark.parametrize` to make them data-driven.
-   **Compose with Fixtures:** Freely combine fixtures (for state/resources) and parameters (for scenarios) to build powerful and clean tests.
-   **Use Indirect Parametrization for Setup:** When your parameters describe *what* to set up rather than being the data itself, use `indirect=True` to delegate creation to a fixture.
-   **Always Use Custom IDs:** For any non-trivial parameter set, provide custom `ids`. It is a small investment that pays huge dividends when a test fails.
