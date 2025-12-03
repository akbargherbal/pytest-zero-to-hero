# Chapter 5: Parameterized Testing

## Testing Multiple Scenarios Without Repetition

## The Problem: Copy-Paste Testing

You're building a password validation system. The requirements are clear:

- Passwords must be at least 8 characters
- Passwords must contain at least one uppercase letter
- Passwords must contain at least one number
- Passwords must contain at least one special character

You need to test all these rules. Your first instinct might be to write separate test functions for each scenario.

```python
# password_validator.py
def validate_password(password: str) -> tuple[bool, str]:
    """
    Validate a password against security requirements.
    
    Returns:
        (is_valid, error_message) tuple
    """
    if len(password) < 8:
        return False, "Password must be at least 8 characters"
    
    if not any(c.isupper() for c in password):
        return False, "Password must contain at least one uppercase letter"
    
    if not any(c.isdigit() for c in password):
        return False, "Password must contain at least one number"
    
    if not any(c in "!@#$%^&*()_+-=[]{}|;:,.<>?" for c in password):
        return False, "Password must contain at least one special character"
    
    return True, "Password is valid"
```

### Phase 1: The Naive Approach - Separate Test Functions

Let's write tests the straightforward way: one test function per scenario.

```python
# test_password_validator_naive.py
from password_validator import validate_password

def test_password_too_short():
    is_valid, message = validate_password("Short1!")
    assert not is_valid
    assert message == "Password must be at least 8 characters"

def test_password_no_uppercase():
    is_valid, message = validate_password("password123!")
    assert not is_valid
    assert message == "Password must contain at least one uppercase letter"

def test_password_no_number():
    is_valid, message = validate_password("Password!")
    assert not is_valid
    assert message == "Password must contain at least one number"

def test_password_no_special_char():
    is_valid, message = validate_password("Password123")
    assert not is_valid
    assert message == "Password must contain at least one special character"

def test_password_valid():
    is_valid, message = validate_password("Password123!")
    assert is_valid
    assert message == "Password is valid"
```

Let's run these tests to verify they work:

```bash
pytest test_password_validator_naive.py -v
```

**Output**:

```text
test_password_validator_naive.py::test_password_too_short PASSED
test_password_validator_naive.py::test_password_no_uppercase PASSED
test_password_validator_naive.py::test_password_no_number PASSED
test_password_validator_naive.py::test_password_no_special_char PASSED
test_password_validator_naive.py::test_password_valid PASSED

======================== 5 passed in 0.02s =========================
```

Great! All tests pass. But look at the code. Notice anything?

### The Hidden Cost of Repetition

Every test function follows the exact same pattern:

1. Call `validate_password()` with a test input
2. Assert the validity boolean
3. Assert the error message

The only things that change are:
- The input password
- The expected validity
- The expected message

**This is the anchor example we'll refine throughout this chapter.** We have working tests, but they suffer from severe code duplication.

### Current Limitations

**What happens when requirements change?**

Imagine your product manager says: "Actually, we need to return a structured error object instead of a tuple."

You'd need to modify **every single test function**. Five tests means five places to update. In a real system with dozens of validation rules, this becomes:

- **Maintenance nightmare**: Change the function signature? Update 50 test functions.
- **Error-prone**: Miss one test during refactoring? Silent bugs.
- **Verbose**: 100 lines of code to test what's essentially a data table.
- **Hard to extend**: Adding a new test case means writing an entire new function.

### What We Need

We need a way to:
1. Define the test logic **once**
2. Run it against **multiple inputs**
3. Get **separate test results** for each input
4. Keep tests **readable and maintainable**

This is exactly what **parameterized testing** solves.

## @pytest.mark.parametrize Basics

## Iteration 1: Introducing Parametrization

Let's transform our repetitive tests into a single parameterized test. We'll start with the simplest possible example to understand the mechanism.

### The Basic Syntax

The `@pytest.mark.parametrize` decorator takes two arguments:

1. **Parameter names** (as a string)
2. **Parameter values** (as a list)

Here's the minimal transformation:

```python
# test_password_validator_param_v1.py
import pytest
from password_validator import validate_password

@pytest.mark.parametrize("password", [
    "Short1!",
    "password123!",
    "Password!",
    "Password123",
    "Password123!",
])
def test_password_validation(password):
    is_valid, message = validate_password(password)
    # What do we assert here?
```

Wait. We have a problem. We're passing different passwords, but we need to check different **expected results** for each one. The test above is incomplete—we can't write meaningful assertions without knowing what each password should produce.

### Diagnostic Analysis: The Incomplete Parametrization

Let's try to run this incomplete test to see what happens:

```bash
pytest test_password_validator_param_v1.py -v
```

**Output**:

```text
test_password_validator_param_v1.py::test_password_validation[Short1!] PASSED
test_password_validator_param_v1.py::test_password_validation[password123!] PASSED
test_password_validator_param_v1.py::test_password_validation[Password!] PASSED
test_password_validator_param_v1.py::test_password_validation[Password123] PASSED
test_password_validator_param_v1.py::test_password_validation[Password123!] PASSED

======================== 5 passed in 0.02s =========================
```

**The tests pass, but they're meaningless.** Without assertions, pytest just executes the function and considers it passed if no exception is raised.

Notice something important in the output: `test_password_validation[Short1!]`, `test_password_validation[password123!]`, etc. Pytest automatically:

1. **Generated 5 separate test cases** from our single test function
2. **Named each test** by appending the parameter value in brackets
3. **Ran each test independently**

This is the core mechanism of parametrization. But we need to pass **both** the input and the expected output.

### Iteration 2: Parametrizing Multiple Values

To make our test meaningful, we need to pass tuples containing both the input and expected results:

```python
# test_password_validator_param_v2.py
import pytest
from password_validator import validate_password

@pytest.mark.parametrize("password,expected_valid,expected_message", [
    ("Short1!", False, "Password must be at least 8 characters"),
    ("password123!", False, "Password must contain at least one uppercase letter"),
    ("Password!", False, "Password must contain at least one number"),
    ("Password123", False, "Password must contain at least one special character"),
    ("Password123!", True, "Password is valid"),
])
def test_password_validation(password, expected_valid, expected_message):
    is_valid, message = validate_password(password)
    assert is_valid == expected_valid
    assert message == expected_message
```

**Key changes**:

1. **Parameter names**: `"password,expected_valid,expected_message"` - comma-separated string
2. **Parameter values**: List of tuples, each containing three values
3. **Test function signature**: Now accepts three parameters matching the names
4. **Assertions**: Compare actual results against expected values

Let's run this:

```bash
pytest test_password_validator_param_v2.py -v
```

**Output**:

```text
test_password_validator_param_v2.py::test_password_validation[Short1!-False-Password must be at least 8 characters] PASSED
test_password_validator_param_v2.py::test_password_validation[password123!-False-Password must contain at least one uppercase letter] PASSED
test_password_validator_param_v2.py::test_password_validation[Password!-False-Password must contain at least one number] PASSED
test_password_validator_param_v2.py::test_password_validation[Password123-False-Password must contain at least one special character] PASSED
test_password_validator_param_v2.py::test_password_validation[Password123!-True-Password is valid] PASSED

======================== 5 passed in 0.02s =========================
```

**Success!** We've transformed 5 separate test functions into a single parameterized test.

### Understanding What Just Happened

Pytest performed these steps:

1. **Read the decorator**: Found `@pytest.mark.parametrize` with parameter names and values
2. **Generated test cases**: Created 5 test instances, one per tuple in the list
3. **Unpacked parameters**: For each test, unpacked the tuple into the three named parameters
4. **Executed independently**: Ran each test case as if it were a separate function
5. **Reported separately**: Each test case appears as a distinct line in the output

### The Mechanics: How Parametrization Works

When pytest encounters `@pytest.mark.parametrize`, it essentially does this transformation internally:

**Before (what you write)**:

```python
@pytest.mark.parametrize("x,y", [(1, 2), (3, 4)])
def test_addition(x, y):
    assert x + y > 0
```

**After (what pytest executes)**:

```python
def test_addition_case_0():
    x, y = 1, 2
    assert x + y > 0

def test_addition_case_1():
    x, y = 3, 4
    assert x + y > 0
```

Each parameterized test is a **completely independent test execution**. If one fails, the others still run.

### Demonstrating Independence: Intentional Failure

Let's prove that test cases are independent by introducing a failing case:

```python
# test_password_validator_param_v3.py
import pytest
from password_validator import validate_password

@pytest.mark.parametrize("password,expected_valid,expected_message", [
    ("Short1!", False, "Password must be at least 8 characters"),
    ("password123!", False, "WRONG MESSAGE"),  # Intentionally wrong
    ("Password!", False, "Password must contain at least one number"),
    ("Password123", False, "Password must contain at least one special character"),
    ("Password123!", True, "Password is valid"),
])
def test_password_validation(password, expected_valid, expected_message):
    is_valid, message = validate_password(password)
    assert is_valid == expected_valid
    assert message == expected_message
```

```bash
pytest test_password_validator_param_v3.py -v
```

**Output**:

```text
test_password_validator_param_v3.py::test_password_validation[Short1!-False-Password must be at least 8 characters] PASSED
test_password_validator_param_v3.py::test_password_validation[password123!-False-WRONG MESSAGE] FAILED
test_password_validator_param_v3.py::test_password_validation[Password!-False-Password must contain at least one number] PASSED
test_password_validator_param_v3.py::test_password_validation[Password123-False-Password must contain at least one special character] PASSED
test_password_validator_param_v3.py::test_password_validation[Password123!-True-Password is valid] PASSED

================================ FAILURES =================================
____________ test_password_validation[password123!-False-WRONG MESSAGE] ____________

password = 'password123!', expected_valid = False, expected_message = 'WRONG MESSAGE'

    @pytest.mark.parametrize("password,expected_valid,expected_message", [
        ("Short1!", False, "Password must be at least 8 characters"),
        ("password123!", False, "WRONG MESSAGE"),
        ("Password!", False, "Password must contain at least one number"),
        ("Password123", False, "Password must contain at least one special character"),
        ("Password123!", True, "Password is valid"),
    ])
    def test_password_validation(password, expected_valid, expected_message):
        is_valid, message = validate_password(password)
        assert is_valid == expected_valid
>       assert message == expected_message
E       AssertionError: assert 'Password must...ppercase letter' == 'WRONG MESSAGE'
E         - WRONG MESSAGE
E         + Password must contain at least one uppercase letter

test_password_validator_param_v3.py:13: AssertionError
==================== 1 failed, 4 passed in 0.03s ====================
```

### Diagnostic Analysis: Reading the Parametrized Failure

**The summary line**:
```
test_password_validation[password123!-False-WRONG MESSAGE] FAILED
```

This tells us:
- **Which test function**: `test_password_validation`
- **Which parameter set**: `[password123!-False-WRONG MESSAGE]` - the second tuple in our list
- **The result**: `FAILED`

**The failure details**:
```python
password = 'password123!', expected_valid = False, expected_message = 'WRONG MESSAGE'
```

Pytest shows us the **exact parameter values** that caused this specific test case to fail. This is crucial for debugging—you immediately know which scenario broke.

**The assertion introspection**:
```
E       AssertionError: assert 'Password must...ppercase letter' == 'WRONG MESSAGE'
E         - WRONG MESSAGE
E         + Password must contain at least one uppercase letter
```

Pytest shows:
- What we expected: `'WRONG MESSAGE'`
- What we got: `'Password must contain at least one uppercase letter'`

**Key insight**: The other 4 test cases continued running and passed. One failing parameter set doesn't stop the entire test function.

### Comparison: Before and After

**Before parametrization** (5 functions, 25 lines):

```python
def test_password_too_short():
    is_valid, message = validate_password("Short1!")
    assert not is_valid
    assert message == "Password must be at least 8 characters"

def test_password_no_uppercase():
    is_valid, message = validate_password("password123!")
    assert not is_valid
    assert message == "Password must contain at least one uppercase letter"

# ... 3 more functions
```

**After parametrization** (1 function, 13 lines):

```python
@pytest.mark.parametrize("password,expected_valid,expected_message", [
    ("Short1!", False, "Password must be at least 8 characters"),
    ("password123!", False, "Password must contain at least one uppercase letter"),
    ("Password!", False, "Password must contain at least one number"),
    ("Password123", False, "Password must contain at least one special character"),
    ("Password123!", True, "Password is valid"),
])
def test_password_validation(password, expected_valid, expected_message):
    is_valid, message = validate_password(password)
    assert is_valid == expected_valid
    assert message == expected_message
```

**Benefits achieved**:
- **48% less code**: 13 lines vs 25 lines
- **Single source of truth**: Test logic defined once
- **Easy to extend**: Add new test case = add one line to the list
- **Maintainable**: Change assertion logic in one place
- **Clear data structure**: Test cases are now a readable table

### Current Limitation

Our test names are getting unwieldy:
```
test_password_validation[Short1!-False-Password must be at least 8 characters]
```

This works, but it's verbose and hard to scan. In section 5.6, we'll learn how to generate custom test IDs for better readability.

But first, we need to understand how to parametrize multiple parameters independently, which opens up powerful testing patterns.

## Multiple Parameters

## The Power of Cartesian Products

So far, we've parametrized a single test function with multiple test cases. But what if you need to test **combinations** of different parameters?

### Phase 1: The New Scenario - Email Validation

Let's extend our authentication system with email validation. Emails must:

1. Contain exactly one `@` symbol
2. Have a domain with at least one dot
3. Not start or end with special characters

We want to test this against multiple email formats **and** multiple validation modes (strict vs. lenient).

```python
# email_validator.py
def validate_email(email: str, strict: bool = True) -> tuple[bool, str]:
    """
    Validate an email address.
    
    Args:
        email: Email address to validate
        strict: If True, apply stricter validation rules
        
    Returns:
        (is_valid, error_message) tuple
    """
    if email.count("@") != 1:
        return False, "Email must contain exactly one @ symbol"
    
    local, domain = email.split("@")
    
    if not local or not domain:
        return False, "Email must have both local and domain parts"
    
    if "." not in domain:
        return False, "Domain must contain at least one dot"
    
    if strict:
        # Strict mode: no special characters at start/end
        if local[0] in ".-_" or local[-1] in ".-_":
            return False, "Local part cannot start or end with special characters"
    
    return True, "Email is valid"
```

### Naive Approach: Nested Loops in Your Head

You might think: "I need to test 4 email formats × 2 modes = 8 test cases." Your first instinct might be to write them all out:

```python
# test_email_validator_naive.py
from email_validator import validate_email

def test_valid_email_strict():
    is_valid, _ = validate_email("user@example.com", strict=True)
    assert is_valid

def test_valid_email_lenient():
    is_valid, _ = validate_email("user@example.com", strict=False)
    assert is_valid

def test_email_with_leading_dot_strict():
    is_valid, _ = validate_email(".user@example.com", strict=True)
    assert not is_valid

def test_email_with_leading_dot_lenient():
    is_valid, _ = validate_email(".user@example.com", strict=False)
    assert is_valid

# ... 4 more test functions for other email formats
```

This is tedious and error-prone. We're manually creating the Cartesian product of (email formats) × (validation modes).

### Iteration 1: Stacking Parametrize Decorators

Pytest allows you to **stack multiple `@pytest.mark.parametrize` decorators**. When you do this, pytest automatically generates the Cartesian product of all parameter combinations.

```python
# test_email_validator_param_v1.py
import pytest
from email_validator import validate_email

@pytest.mark.parametrize("strict", [True, False])
@pytest.mark.parametrize("email,should_pass", [
    ("user@example.com", True),
    (".user@example.com", False),
    ("user.@example.com", False),
    ("user@example", False),
])
def test_email_validation(email, should_pass, strict):
    is_valid, message = validate_email(email, strict=strict)
    
    if should_pass and strict:
        assert is_valid, f"Expected {email} to be valid in strict mode"
    elif should_pass and not strict:
        assert is_valid, f"Expected {email} to be valid in lenient mode"
    # ... more conditional logic
```

Wait. This is getting complicated. We have conditional logic in our test, which is a code smell. Let's see what happens when we run this:

```bash
pytest test_email_validator_param_v1.py -v
```

**Output**:

```text
test_email_validator_param_v1.py::test_email_validation[user@example.com-True-True] PASSED
test_email_validator_param_v1.py::test_email_validation[user@example.com-True-False] PASSED
test_email_validator_param_v1.py::test_email_validation[.user@example.com-False-True] PASSED
test_email_validator_param_v1.py::test_email_validation[.user@example.com-False-False] FAILED
test_email_validator_param_v1.py::test_email_validation[user.@example.com-False-True] PASSED
test_email_validator_param_v1.py::test_email_validation[user.@example.com-False-False] FAILED
test_email_validator_param_v1.py::test_email_validation[user@example-False-True] PASSED
test_email_validator_param_v1.py::test_email_validation[user@example-False-False] PASSED

================================ FAILURES =================================
```

### Diagnostic Analysis: The Cartesian Product Problem

**What happened**: Pytest generated **8 test cases** (4 emails × 2 modes), but our test logic is wrong. The problem is that `should_pass` doesn't account for the `strict` parameter.

For example:
- `.user@example.com` should **fail** in strict mode (leading dot)
- `.user@example.com` should **pass** in lenient mode (lenient allows it)

But we marked it as `should_pass=False` unconditionally, so the lenient mode test fails.

**Root cause**: We're trying to use a single boolean (`should_pass`) to represent behavior that depends on **two** variables (email format **and** validation mode).

**What we need**: A way to specify expected results for each **combination** of parameters, not just for each email in isolation.

### Iteration 2: Explicit Combination Parametrization

Instead of stacking decorators, let's explicitly define the combinations we want to test:

```python
# test_email_validator_param_v2.py
import pytest
from email_validator import validate_email

@pytest.mark.parametrize("email,strict,expected_valid,expected_message", [
    # Valid emails pass in both modes
    ("user@example.com", True, True, "Email is valid"),
    ("user@example.com", False, True, "Email is valid"),
    
    # Leading dot: fails strict, passes lenient
    (".user@example.com", True, False, "Local part cannot start or end with special characters"),
    (".user@example.com", False, True, "Email is valid"),
    
    # Trailing dot: fails strict, passes lenient
    ("user.@example.com", True, False, "Local part cannot start or end with special characters"),
    ("user.@example.com", False, True, "Email is valid"),
    
    # Missing domain dot: fails both modes
    ("user@example", True, False, "Domain must contain at least one dot"),
    ("user@example", False, False, "Domain must contain at least one dot"),
])
def test_email_validation(email, strict, expected_valid, expected_message):
    is_valid, message = validate_email(email, strict=strict)
    assert is_valid == expected_valid
    assert message == expected_message
```

```bash
pytest test_email_validator_param_v2.py -v
```

**Output**:

```text
test_email_validator_param_v2.py::test_email_validation[user@example.com-True-True-Email is valid] PASSED
test_email_validator_param_v2.py::test_email_validation[user@example.com-False-True-Email is valid] PASSED
test_email_validator_param_v2.py::test_email_validation[.user@example.com-True-False-Local part cannot start or end with special characters] PASSED
test_email_validator_param_v2.py::test_email_validation[.user@example.com-False-True-Email is valid] PASSED
test_email_validator_param_v2.py::test_email_validation[user.@example.com-True-False-Local part cannot start or end with special characters] PASSED
test_email_validator_param_v2.py::test_email_validation[user.@example.com-False-True-Email is valid] PASSED
test_email_validator_param_v2.py::test_email_validation[user@example-True-False-Domain must contain at least one dot] PASSED
test_email_validator_param_v2.py::test_email_validation[user@example-False-False-Domain must contain at least one dot] PASSED

======================== 8 passed in 0.03s =========================
```

**Success!** All 8 test cases pass. But notice we had to manually write out all 8 combinations.

### When to Use Stacked Decorators vs. Explicit Combinations

**Use stacked decorators when**:
- The parameters are **independent**
- You want to test **all combinations**
- The expected result is the **same** for all combinations

**Example**: Testing a function with different input types and different operations:

```python
@pytest.mark.parametrize("operation", ["add", "subtract", "multiply"])
@pytest.mark.parametrize("input_type", [int, float, complex])
def test_calculator(operation, input_type):
    # Test that calculator works with all combinations
    # of operations and input types
    pass
```

This generates 3 × 3 = 9 test cases automatically.

**Use explicit combinations when**:
- The expected result **depends on the combination**
- Not all combinations are valid or meaningful
- You need fine-grained control over test cases

**Example**: Our email validation where strict/lenient mode changes the expected result.

### Iteration 3: Hybrid Approach - Stacking with Conditional Logic

Sometimes you want the convenience of stacking but need some conditional behavior. Here's a pattern that works:

```python
# test_email_validator_param_v3.py
import pytest
from email_validator import validate_email

# Define test cases with their strict-mode behavior
EMAIL_TEST_CASES = [
    ("user@example.com", True, "Email is valid"),  # Valid in both modes
    (".user@example.com", False, "Local part cannot start or end with special characters"),  # Strict-only failure
    ("user.@example.com", False, "Local part cannot start or end with special characters"),  # Strict-only failure
    ("user@example", False, "Domain must contain at least one dot"),  # Fails in both modes
]

@pytest.mark.parametrize("strict", [True, False])
@pytest.mark.parametrize("email,strict_fails,strict_message", EMAIL_TEST_CASES)
def test_email_validation(email, strict_fails, strict_message, strict):
    is_valid, message = validate_email(email, strict=strict)
    
    if strict:
        # In strict mode, use the provided expectations
        assert is_valid == (not strict_fails)
        if strict_fails:
            assert message == strict_message
    else:
        # In lenient mode, only domain-level failures matter
        if "Domain" in strict_message:
            assert not is_valid
            assert message == strict_message
        else:
            assert is_valid
            assert message == "Email is valid"
```

```bash
pytest test_email_validator_param_v3.py -v
```

**Output**:

```text
test_email_validator_param_v3.py::test_email_validation[True-user@example.com-True-Email is valid] PASSED
test_email_validator_param_v3.py::test_email_validation[True-.user@example.com-False-Local part cannot start or end with special characters] PASSED
test_email_validator_param_v3.py::test_email_validation[True-user.@example.com-False-Local part cannot start or end with special characters] PASSED
test_email_validator_param_v3.py::test_email_validation[True-user@example-False-Domain must contain at least one dot] PASSED
test_email_validator_param_v3.py::test_email_validation[False-user@example.com-True-Email is valid] PASSED
test_email_validator_param_v3.py::test_email_validation[False-.user@example.com-False-Local part cannot start or end with special characters] PASSED
test_email_validator_param_v3.py::test_email_validation[False-user.@example.com-False-Local part cannot start or end with special characters] PASSED
test_email_validator_param_v3.py::test_email_validation[False-user@example-False-Domain must contain at least one dot] PASSED

======================== 8 passed in 0.03s =========================
```

This approach:
- ✅ Generates all combinations automatically (4 emails × 2 modes = 8 tests)
- ✅ Defines email test cases once
- ⚠️ Contains conditional logic in the test (acceptable when the logic is simple and clear)

### When to Apply Multiple Parameter Approaches

| Scenario | Approach | Reason |
|----------|----------|--------|
| Parameters are independent, same expected result | Stacked decorators | Automatic Cartesian product |
| Expected result depends on combination | Explicit combinations | Full control over expectations |
| Some conditional logic, but mostly independent | Hybrid (stacked + conditionals) | Balance between DRY and clarity |
| Many parameters, few meaningful combinations | Explicit combinations | Avoid testing invalid combinations |

### Current Limitation

Our test names are still auto-generated and verbose:
```
test_email_validation[True-user@example.com-True-Email is valid]
```

This is functional but not ideal for quickly scanning test results. We'll address this in section 5.6 with custom test IDs.

But first, we need to understand how parametrization interacts with fixtures—a powerful combination that enables sophisticated test setups.

## Combining Parametrization with Fixtures

## When Test Data Needs Setup

So far, our parametrized tests have used simple values: strings, booleans, integers. But what if your test data requires **setup and teardown**? What if you need to test against database connections, file handles, or API clients?

This is where parametrization meets fixtures.

### Phase 1: The New Scenario - Testing Database Queries

Let's build a user repository that queries a database. We want to test it against multiple database backends: SQLite, PostgreSQL, and MySQL.

```python
# user_repository.py
class UserRepository:
    """Repository for user data operations."""
    
    def __init__(self, db_connection):
        self.db = db_connection
    
    def create_user(self, username: str, email: str) -> int:
        """Create a user and return their ID."""
        cursor = self.db.cursor()
        cursor.execute(
            "INSERT INTO users (username, email) VALUES (?, ?)",
            (username, email)
        )
        self.db.commit()
        return cursor.lastrowid
    
    def get_user(self, user_id: int) -> dict:
        """Retrieve a user by ID."""
        cursor = self.db.cursor()
        cursor.execute("SELECT id, username, email FROM users WHERE id = ?", (user_id,))
        row = cursor.fetchone()
        if row:
            return {"id": row[0], "username": row[1], "email": row[2]}
        return None
    
    def count_users(self) -> int:
        """Count total users."""
        cursor = self.db.cursor()
        cursor.execute("SELECT COUNT(*) FROM users")
        return cursor.fetchone()[0]
```

### Naive Approach: Separate Test Functions Per Database

You might write separate test functions for each database:

```python
# test_user_repository_naive.py
import sqlite3
import pytest
from user_repository import UserRepository

def test_create_user_sqlite():
    # Setup SQLite
    conn = sqlite3.connect(":memory:")
    conn.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT, email TEXT)")
    
    repo = UserRepository(conn)
    user_id = repo.create_user("alice", "alice@example.com")
    
    assert user_id > 0
    assert repo.count_users() == 1
    
    conn.close()

def test_create_user_postgres():
    # Setup PostgreSQL
    # ... connection code ...
    pass

def test_create_user_mysql():
    # Setup MySQL
    # ... connection code ...
    pass
```

This is repetitive. The test logic is identical—only the database connection changes. We need to parametrize the **database connection**, but connections require setup and teardown.

### Iteration 1: Fixture Without Parametrization

First, let's extract the database setup into a fixture:

```python
# test_user_repository_fixture_v1.py
import sqlite3
import pytest
from user_repository import UserRepository

@pytest.fixture
def db_connection():
    """Provide an in-memory SQLite database."""
    conn = sqlite3.connect(":memory:")
    conn.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT, email TEXT)")
    yield conn
    conn.close()

def test_create_user(db_connection):
    repo = UserRepository(db_connection)
    user_id = repo.create_user("alice", "alice@example.com")
    
    assert user_id > 0
    assert repo.count_users() == 1

def test_get_user(db_connection):
    repo = UserRepository(db_connection)
    user_id = repo.create_user("bob", "bob@example.com")
    
    user = repo.get_user(user_id)
    assert user["username"] == "bob"
    assert user["email"] == "bob@example.com"
```

```bash
pytest test_user_repository_fixture_v1.py -v
```

**Output**:

```text
test_user_repository_fixture_v1.py::test_create_user PASSED
test_user_repository_fixture_v1.py::test_get_user PASSED

======================== 2 passed in 0.02s =========================
```

Good! But we're only testing SQLite. How do we test against multiple databases?

### Iteration 2: Parametrizing the Fixture

Pytest allows you to **parametrize fixtures** using the `params` argument to `@pytest.fixture`. This is the key to testing against multiple setups.

```python
# test_user_repository_fixture_v2.py
import sqlite3
import pytest
from user_repository import UserRepository

@pytest.fixture(params=["sqlite", "postgres", "mysql"])
def db_connection(request):
    """Provide database connections for multiple backends."""
    db_type = request.param
    
    if db_type == "sqlite":
        conn = sqlite3.connect(":memory:")
        conn.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT, email TEXT)")
        yield conn
        conn.close()
    
    elif db_type == "postgres":
        # For demonstration, we'll simulate PostgreSQL
        # In real code, you'd use psycopg2
        conn = sqlite3.connect(":memory:")  # Simulating
        conn.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT, email TEXT)")
        yield conn
        conn.close()
    
    elif db_type == "mysql":
        # For demonstration, we'll simulate MySQL
        # In real code, you'd use mysql-connector-python
        conn = sqlite3.connect(":memory:")  # Simulating
        conn.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT, email TEXT)")
        yield conn
        conn.close()

def test_create_user(db_connection):
    repo = UserRepository(db_connection)
    user_id = repo.create_user("alice", "alice@example.com")
    
    assert user_id > 0
    assert repo.count_users() == 1

def test_get_user(db_connection):
    repo = UserRepository(db_connection)
    user_id = repo.create_user("bob", "bob@example.com")
    
    user = repo.get_user(user_id)
    assert user["username"] == "bob"
    assert user["email"] == "bob@example.com"
```

```bash
pytest test_user_repository_fixture_v2.py -v
```

**Output**:

```text
test_user_repository_fixture_v2.py::test_create_user[sqlite] PASSED
test_user_repository_fixture_v2.py::test_create_user[postgres] PASSED
test_user_repository_fixture_v2.py::test_create_user[mysql] PASSED
test_user_repository_fixture_v2.py::test_get_user[sqlite] PASSED
test_user_repository_fixture_v2.py::test_get_user[postgres] PASSED
test_user_repository_fixture_v2.py::test_get_user[mysql] PASSED

======================== 6 passed in 0.03s =========================
```

### Understanding What Just Happened

**The mechanics**:

1. **`params=["sqlite", "postgres", "mysql"]`**: Tells pytest to run the fixture three times, once per parameter
2. **`request.param`**: Inside the fixture, this gives you the current parameter value
3. **Test multiplication**: Each test function that uses `db_connection` runs **three times** (once per database)

We wrote **2 test functions** but got **6 test executions** (2 tests × 3 databases).

### The Power of Fixture Parametrization

This pattern is incredibly powerful because:

1. **Test logic stays DRY**: Write the test once, run it against multiple setups
2. **Setup is encapsulated**: Database connection logic lives in the fixture
3. **Automatic cleanup**: The `yield` pattern ensures proper teardown
4. **Scales effortlessly**: Add a new database? Add one line to `params`

### Combining Test Parametrization with Fixture Parametrization

Now let's combine both techniques. We'll parametrize the test **and** the fixture:

```python
# test_user_repository_combined.py
import sqlite3
import pytest
from user_repository import UserRepository

@pytest.fixture(params=["sqlite", "postgres"])
def db_connection(request):
    """Provide database connections for multiple backends."""
    db_type = request.param
    
    if db_type == "sqlite":
        conn = sqlite3.connect(":memory:")
        conn.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT, email TEXT)")
        yield conn
        conn.close()
    
    elif db_type == "postgres":
        conn = sqlite3.connect(":memory:")  # Simulating
        conn.execute("CREATE TABLE users (id INTEGER PRIMARY KEY, username TEXT, email TEXT)")
        yield conn
        conn.close()

@pytest.mark.parametrize("username,email", [
    ("alice", "alice@example.com"),
    ("bob", "bob@example.com"),
    ("charlie", "charlie@example.com"),
])
def test_create_user(db_connection, username, email):
    repo = UserRepository(db_connection)
    user_id = repo.create_user(username, email)
    
    assert user_id > 0
    
    user = repo.get_user(user_id)
    assert user["username"] == username
    assert user["email"] == email
```

```bash
pytest test_user_repository_combined.py -v
```

**Output**:

```text
test_user_repository_combined.py::test_create_user[sqlite-alice-alice@example.com] PASSED
test_user_repository_combined.py::test_create_user[sqlite-bob-bob@example.com] PASSED
test_user_repository_combined.py::test_create_user[sqlite-charlie-charlie@example.com] PASSED
test_user_repository_combined.py::test_create_user[postgres-alice-alice@example.com] PASSED
test_user_repository_combined.py::test_create_user[postgres-bob-bob@example.com] PASSED
test_user_repository_combined.py::test_create_user[postgres-charlie-charlie@example.com] PASSED

======================== 6 passed in 0.03s =========================
```

**The multiplication**:
- 2 database backends (fixture params)
- × 3 user test cases (test params)
- = **6 total test executions**

We wrote **1 test function** and got **6 comprehensive test cases** covering multiple databases and multiple user scenarios.

### Demonstrating Failure Isolation

Let's introduce a bug that only affects one database backend:

```python
# user_repository_buggy.py
class UserRepository:
    def __init__(self, db_connection):
        self.db = db_connection
    
    def create_user(self, username: str, email: str) -> int:
        cursor = self.db.cursor()
        
        # Bug: PostgreSQL uses different placeholder syntax
        # This will fail for postgres but work for sqlite
        cursor.execute(
            "INSERT INTO users (username, email) VALUES (?, ?)",
            (username, email)
        )
        self.db.commit()
        return cursor.lastrowid
```

If we had a real PostgreSQL connection (which uses `%s` placeholders instead of `?`), we'd see:

```text
test_user_repository_combined.py::test_create_user[sqlite-alice-alice@example.com] PASSED
test_user_repository_combined.py::test_create_user[sqlite-bob-bob@example.com] PASSED
test_user_repository_combined.py::test_create_user[sqlite-charlie-charlie@example.com] PASSED
test_user_repository_combined.py::test_create_user[postgres-alice-alice@example.com] FAILED
test_user_repository_combined.py::test_create_user[postgres-bob-bob@example.com] FAILED
test_user_repository_combined.py::test_create_user[postgres-charlie-charlie@example.com] FAILED

================================ FAILURES =================================
____________ test_create_user[postgres-alice-alice@example.com] ____________
...
ProgrammingError: syntax error at or near "?"
```

**Key insight**: The failure is **isolated to the postgres parameter**. All SQLite tests pass. This immediately tells you the bug is database-specific, not a logic error in your test.

### When to Apply This Pattern

**Use fixture parametrization when**:
- You need to test against multiple **environments** (databases, APIs, file systems)
- Setup/teardown is **complex** and shouldn't be in the test
- You want to ensure **cross-platform compatibility**

**Use combined parametrization when**:
- You need to test multiple **scenarios** across multiple **environments**
- You want **comprehensive coverage** with minimal code
- The test logic is **identical** across all combinations

**Avoid when**:
- Different environments require **different test logic** (use separate test functions)
- The number of combinations is **excessive** (2 fixtures × 10 params each = 20 tests might be too many)
- Setup is **trivial** (just use test parametrization)

### Current Limitation

Our test IDs are getting very long:
```
test_create_user[postgres-alice-alice@example.com]
```

When you have multiple parameters, the auto-generated IDs become unwieldy. We need a way to create **custom, readable test IDs**.

But first, there's one more advanced parametrization technique to cover: **indirect parametrization**, which gives you even more control over how parameters flow into fixtures.

## Indirect Parametrization

## When Parameters Need Transformation

So far, we've seen two parametrization patterns:

1. **Direct parametrization**: Parameters go straight to the test function
2. **Fixture parametrization**: Parameters are defined in the fixture itself

But what if you need a **hybrid**? What if you want to:
- Define parameter values in the test (for visibility)
- But have the fixture **transform** those values before use

This is **indirect parametrization**.

### Phase 1: The New Scenario - Testing with Different File Formats

Let's build a data loader that reads configuration from files. We want to test it with JSON, YAML, and TOML formats.

```python
# config_loader.py
import json
import yaml
import tomli

class ConfigLoader:
    """Load configuration from various file formats."""
    
    def __init__(self, file_path: str):
        self.file_path = file_path
        self.config = self._load()
    
    def _load(self) -> dict:
        """Load config based on file extension."""
        if self.file_path.endswith(".json"):
            with open(self.file_path) as f:
                return json.load(f)
        elif self.file_path.endswith(".yaml") or self.file_path.endswith(".yml"):
            with open(self.file_path) as f:
                return yaml.safe_load(f)
        elif self.file_path.endswith(".toml"):
            with open(self.file_path, "rb") as f:
                return tomli.load(f)
        else:
            raise ValueError(f"Unsupported file format: {self.file_path}")
    
    def get(self, key: str, default=None):
        """Get a configuration value."""
        return self.config.get(key, default)
```

### Naive Approach: Fixture Creates All Files

You might create a fixture that generates all file formats:

```python
# test_config_loader_naive.py
import pytest
import json
import yaml
import tomli_w
from pathlib import Path
from config_loader import ConfigLoader

@pytest.fixture
def config_files(tmp_path):
    """Create config files in all formats."""
    config_data = {"app_name": "TestApp", "version": "1.0", "debug": True}
    
    # Create JSON file
    json_file = tmp_path / "config.json"
    json_file.write_text(json.dumps(config_data))
    
    # Create YAML file
    yaml_file = tmp_path / "config.yaml"
    yaml_file.write_text(yaml.dump(config_data))
    
    # Create TOML file
    toml_file = tmp_path / "config.toml"
    with open(toml_file, "wb") as f:
        tomli_w.dump(config_data, f)
    
    return {
        "json": json_file,
        "yaml": yaml_file,
        "toml": toml_file,
    }

def test_load_json(config_files):
    loader = ConfigLoader(str(config_files["json"]))
    assert loader.get("app_name") == "TestApp"

def test_load_yaml(config_files):
    loader = ConfigLoader(str(config_files["yaml"]))
    assert loader.get("app_name") == "TestApp"

def test_load_toml(config_files):
    loader = ConfigLoader(str(config_files["toml"]))
    assert loader.get("app_name") == "TestApp"
```

**Problems**:
1. The fixture creates **all three files** even if a test only needs one
2. We have **three separate test functions** with identical logic
3. Adding a new format requires modifying the fixture **and** adding a new test function

### Iteration 1: Fixture Parametrization (Review)

We could use fixture parametrization:

```python
# test_config_loader_fixture_param.py
import pytest
import json
import yaml
import tomli_w
from config_loader import ConfigLoader

@pytest.fixture(params=["json", "yaml", "toml"])
def config_file(request, tmp_path):
    """Create a config file in the specified format."""
    file_format = request.param
    config_data = {"app_name": "TestApp", "version": "1.0", "debug": True}
    
    if file_format == "json":
        file_path = tmp_path / "config.json"
        file_path.write_text(json.dumps(config_data))
    elif file_format == "yaml":
        file_path = tmp_path / "config.yaml"
        file_path.write_text(yaml.dump(config_data))
    elif file_format == "toml":
        file_path = tmp_path / "config.toml"
        with open(file_path, "wb") as f:
            tomli_w.dump(config_data, f)
    
    return str(file_path)

def test_load_config(config_file):
    loader = ConfigLoader(config_file)
    assert loader.get("app_name") == "TestApp"
    assert loader.get("version") == "1.0"
    assert loader.get("debug") is True
```

```bash
pytest test_config_loader_fixture_param.py -v
```

**Output**:

```text
test_config_loader_fixture_param.py::test_load_config[json] PASSED
test_config_loader_fixture_param.py::test_load_config[yaml] PASSED
test_config_loader_fixture_param.py::test_load_config[toml] PASSED

======================== 3 passed in 0.03s =========================
```

This works! But there's a limitation: **the parameter values are hidden inside the fixture**. When you look at the test function, you can't see what formats are being tested without reading the fixture code.

### Iteration 2: Indirect Parametrization - Making Parameters Visible

**Indirect parametrization** lets you define parameters at the test level but have the fixture process them. This gives you the best of both worlds:

1. **Visibility**: Parameters are visible in the test decorator
2. **Transformation**: The fixture can transform raw parameters into complex objects

```python
# test_config_loader_indirect.py
import pytest
import json
import yaml
import tomli_w
from config_loader import ConfigLoader

@pytest.fixture
def config_file(request, tmp_path):
    """
    Create a config file in the specified format.
    
    Expects request.param to be the file format string.
    """
    file_format = request.param
    config_data = {"app_name": "TestApp", "version": "1.0", "debug": True}
    
    if file_format == "json":
        file_path = tmp_path / "config.json"
        file_path.write_text(json.dumps(config_data))
    elif file_format == "yaml":
        file_path = tmp_path / "config.yaml"
        file_path.write_text(yaml.dump(config_data))
    elif file_format == "toml":
        file_path = tmp_path / "config.toml"
        with open(file_path, "wb") as f:
            tomli_w.dump(config_data, f)
    else:
        raise ValueError(f"Unsupported format: {file_format}")
    
    return str(file_path)

@pytest.mark.parametrize("config_file", ["json", "yaml", "toml"], indirect=True)
def test_load_config(config_file):
    loader = ConfigLoader(config_file)
    assert loader.get("app_name") == "TestApp"
    assert loader.get("version") == "1.0"
    assert loader.get("debug") is True
```

**Key differences**:

1. **`@pytest.fixture`** (no `params`): The fixture doesn't define its own parameters
2. **`@pytest.mark.parametrize("config_file", [...], indirect=True)`**: 
   - The test defines the parameter values
   - `indirect=True` tells pytest to pass these values to the fixture via `request.param`
3. **Fixture receives `request.param`**: The fixture gets the parameter value and transforms it

Let's run it:

```bash
pytest test_config_loader_indirect.py -v
```

**Output**:

```text
test_config_loader_indirect.py::test_load_config[json] PASSED
test_config_loader_indirect.py::test_load_config[yaml] PASSED
test_config_loader_indirect.py::test_load_config[toml] PASSED

======================== 3 passed in 0.03s =========================
```

### Understanding the Flow

**Without `indirect=True` (direct parametrization)**:
```
Parameter value → Test function directly
```

**With `indirect=True` (indirect parametrization)**:
```
Parameter value → Fixture (via request.param) → Transformed value → Test function
```

### Why This Matters: Complex Object Creation

The real power of indirect parametrization becomes clear when you need to create complex objects. Let's extend our example:

```python
# test_config_loader_indirect_v2.py
import pytest
import json
import yaml
import tomli_w
from config_loader import ConfigLoader

@pytest.fixture
def config_file(request, tmp_path):
    """
    Create a config file based on a specification.
    
    Expects request.param to be a dict with:
        - format: str (json/yaml/toml)
        - data: dict (config data to write)
    """
    spec = request.param
    file_format = spec["format"]
    config_data = spec["data"]
    
    if file_format == "json":
        file_path = tmp_path / "config.json"
        file_path.write_text(json.dumps(config_data))
    elif file_format == "yaml":
        file_path = tmp_path / "config.yaml"
        file_path.write_text(yaml.dump(config_data))
    elif file_format == "toml":
        file_path = tmp_path / "config.toml"
        with open(file_path, "wb") as f:
            tomli_w.dump(config_data, f)
    
    return str(file_path)

@pytest.mark.parametrize("config_file", [
    {"format": "json", "data": {"app_name": "TestApp", "version": "1.0"}},
    {"format": "yaml", "data": {"app_name": "TestApp", "version": "2.0"}},
    {"format": "toml", "data": {"app_name": "ProdApp", "version": "1.0"}},
], indirect=True)
def test_load_config(config_file):
    loader = ConfigLoader(config_file)
    assert loader.get("app_name") in ["TestApp", "ProdApp"]
    assert loader.get("version") in ["1.0", "2.0"]
```

Now we're passing **complex specifications** (dictionaries) as parameters, and the fixture transforms them into actual files. This pattern is incredibly powerful for:

- Creating test databases with specific schemas
- Setting up API clients with different configurations
- Generating test files with varying content

### Partial Indirect Parametrization

You can mix direct and indirect parameters in the same test:

```python
# test_config_loader_mixed.py
import pytest
import json
from config_loader import ConfigLoader

@pytest.fixture
def config_file(request, tmp_path):
    """Create a config file in the specified format."""
    file_format = request.param
    config_data = {"app_name": "TestApp", "version": "1.0"}
    
    file_path = tmp_path / f"config.{file_format}"
    file_path.write_text(json.dumps(config_data))
    return str(file_path)

@pytest.mark.parametrize("config_file", ["json"], indirect=True)
@pytest.mark.parametrize("key,expected", [
    ("app_name", "TestApp"),
    ("version", "1.0"),
    ("debug", None),  # Key doesn't exist
])
def test_get_config_value(config_file, key, expected):
    loader = ConfigLoader(config_file)
    assert loader.get(key) == expected
```

**What's happening**:
- `config_file` is **indirect**: Goes through the fixture
- `key` and `expected` are **direct**: Go straight to the test function

This generates: 1 file format × 3 key-value pairs = **3 test cases**.

```bash
pytest test_config_loader_mixed.py -v
```

**Output**:

```text
test_config_loader_mixed.py::test_get_config_value[json-app_name-TestApp] PASSED
test_config_loader_mixed.py::test_get_config_value[json-version-1.0] PASSED
test_config_loader_mixed.py::test_get_config_value[json-debug-None] PASSED

======================== 3 passed in 0.02s =========================
```

### Demonstrating Failure: Invalid Format

Let's see what happens when we pass an invalid format:

```python
# test_config_loader_indirect_fail.py
import pytest
from config_loader import ConfigLoader

@pytest.fixture
def config_file(request, tmp_path):
    """Create a config file in the specified format."""
    file_format = request.param
    
    if file_format not in ["json", "yaml", "toml"]:
        raise ValueError(f"Unsupported format: {file_format}")
    
    # ... rest of fixture code

@pytest.mark.parametrize("config_file", ["json", "xml", "toml"], indirect=True)
def test_load_config(config_file):
    loader = ConfigLoader(config_file)
    assert loader.get("app_name") == "TestApp"
```

```bash
pytest test_config_loader_indirect_fail.py -v
```

**Output**:

```text
test_config_loader_indirect_fail.py::test_load_config[json] PASSED
test_config_loader_indirect_fail.py::test_load_config[xml] ERROR
test_config_loader_indirect_fail.py::test_load_config[toml] PASSED

================================ ERRORS ===================================
____________ ERROR at setup of test_load_config[xml] ____________

request = <FixtureRequest for <Function test_load_config[xml]>>
tmp_path = PosixPath('/tmp/pytest-of-user/pytest-123/test_load_config_xml0')

    @pytest.fixture
    def config_file(request, tmp_path):
        """Create a config file in the specified format."""
        file_format = request.param
        
        if file_format not in ["json", "yaml", "toml"]:
>           raise ValueError(f"Unsupported format: {file_format}")
E           ValueError: Unsupported format: xml

test_config_loader_indirect_fail.py:8: ValueError
==================== 1 passed, 1 error, 1 passed in 0.03s ====================
```

### Diagnostic Analysis: Fixture Setup Failure

**The summary line**:
```
ERROR at setup of test_load_config[xml]
```

This tells us:
- The error occurred during **fixture setup**, not during the test itself
- The problematic parameter was `xml`

**The error details**:
```python
>           raise ValueError(f"Unsupported format: {file_format}")
E           ValueError: Unsupported format: xml
```

The fixture raised an exception before the test could even run. This is different from a test failure—it's a **setup error**.

**Key insight**: When using indirect parametrization, validation errors in the fixture appear as **setup errors**, not test failures. This helps distinguish between:
- **Setup problems**: The test environment couldn't be created
- **Test failures**: The test ran but assertions failed

### When to Apply Indirect Parametrization

| Scenario | Use Indirect Parametrization | Reason |
|----------|------------------------------|--------|
| Simple values (strings, ints) | ❌ Use direct parametrization | No transformation needed |
| Complex object creation | ✅ Yes | Fixture can build objects from specs |
| File/database setup | ✅ Yes | Fixture handles setup/teardown |
| Parameter validation | ✅ Yes | Fixture can validate and fail early |
| Multiple tests need same transformation | ✅ Yes | DRY: transformation logic in one place |
| Parameters are already the right type | ❌ Use direct parametrization | Simpler and more readable |

### Decision Framework: Direct vs. Indirect

**Use direct parametrization when**:
```python
@pytest.mark.parametrize("value", [1, 2, 3])
def test_something(value):
    assert value > 0
```
- Parameters are **ready to use** as-is
- No setup/teardown needed
- Simple and readable

**Use indirect parametrization when**:
```python
@pytest.mark.parametrize("db", ["sqlite", "postgres"], indirect=True)
def test_something(db):
    # db is a fully configured database connection
    assert db.execute("SELECT 1")
```
- Parameters need **transformation** into complex objects
- Setup/teardown is required
- Multiple tests need the same transformation logic

### Current Limitation

Our test IDs are still auto-generated:
```
test_load_config[json]
test_load_config[yaml]
test_load_config[toml]
```

These are okay, but what if we want more descriptive names? What if we're testing with complex objects and the default representation is unreadable?

This is where **custom test IDs** come in—our final parametrization technique.

## Generating Test IDs for Clarity

## The Problem: Unreadable Test Names

As your parametrized tests grow more complex, the auto-generated test IDs become harder to read. Consider this test:

```python
# test_api_client.py
import pytest

@pytest.mark.parametrize("endpoint,method,headers,expected_status", [
    ("/api/users", "GET", {"Authorization": "Bearer token123"}, 200),
    ("/api/users", "POST", {"Authorization": "Bearer token123", "Content-Type": "application/json"}, 201),
    ("/api/users/123", "DELETE", {"Authorization": "Bearer token123"}, 204),
])
def test_api_request(endpoint, method, headers, expected_status):
    # Test implementation
    pass
```

```bash
pytest test_api_client.py -v
```

**Output**:

```text
test_api_client.py::test_api_request[/api/users-GET-headers0-200] PASSED
test_api_client.py::test_api_request[/api/users-POST-headers1-201] PASSED
test_api_client.py::test_api_request[/api/users/123-DELETE-headers2-204] PASSED
```

**Problems**:
1. `headers0`, `headers1`, `headers2` are meaningless
2. The full parameter values aren't shown (headers are complex)
3. Hard to quickly identify which test failed in CI logs

### Phase 1: Understanding Default ID Generation

Pytest generates test IDs by converting each parameter to a string. For simple types, this works well:

```python
@pytest.mark.parametrize("x", [1, 2, 3])
def test_simple(x):
    pass
# IDs: test_simple[1], test_simple[2], test_simple[3]
```

But for complex types, pytest falls back to generic names:

```python
@pytest.mark.parametrize("data", [{"a": 1}, {"b": 2}])
def test_complex(data):
    pass
# IDs: test_complex[data0], test_complex[data1]  ← Not helpful!
```

### Iteration 1: Using the `ids` Parameter

The `ids` parameter lets you provide custom names for each test case:

```python
# test_api_client_ids_v1.py
import pytest

@pytest.mark.parametrize(
    "endpoint,method,headers,expected_status",
    [
        ("/api/users", "GET", {"Authorization": "Bearer token123"}, 200),
        ("/api/users", "POST", {"Authorization": "Bearer token123", "Content-Type": "application/json"}, 201),
        ("/api/users/123", "DELETE", {"Authorization": "Bearer token123"}, 204),
    ],
    ids=[
        "get_users",
        "create_user",
        "delete_user",
    ]
)
def test_api_request(endpoint, method, headers, expected_status):
    # Test implementation
    pass
```

```bash
pytest test_api_client_ids_v1.py -v
```

**Output**:

```text
test_api_client_ids_v1.py::test_api_request[get_users] PASSED
test_api_client_ids_v1.py::test_api_request[create_user] PASSED
test_api_client_ids_v1.py::test_api_request[delete_user] PASSED
```

**Much better!** Now the test names are meaningful and scannable.

### Iteration 2: Dynamic ID Generation with Functions

Manually writing IDs for every test case is tedious. You can use a **function** to generate IDs automatically:

```python
# test_api_client_ids_v2.py
import pytest

def make_api_test_id(endpoint, method, headers, expected_status):
    """Generate a readable test ID from API parameters."""
    return f"{method}_{endpoint.replace('/', '_')}_{expected_status}"

@pytest.mark.parametrize(
    "endpoint,method,headers,expected_status",
    [
        ("/api/users", "GET", {"Authorization": "Bearer token123"}, 200),
        ("/api/users", "POST", {"Authorization": "Bearer token123", "Content-Type": "application/json"}, 201),
        ("/api/users/123", "DELETE", {"Authorization": "Bearer token123"}, 204),
    ],
    ids=make_api_test_id
)
def test_api_request(endpoint, method, headers, expected_status):
    # Test implementation
    pass
```

```bash
pytest test_api_client_ids_v2.py -v
```

**Output**:

```text
test_api_client_ids_v2.py::test_api_request[GET__api_users_200] PASSED
test_api_client_ids_v2.py::test_api_request[POST__api_users_201] PASSED
test_api_client_ids_v2.py::test_api_request[DELETE__api_users_123_204] PASSED
```

**How it works**:
- Pytest calls `make_api_test_id(endpoint, method, headers, expected_status)` for each parameter set
- The function returns a string that becomes the test ID
- The function receives the **unpacked** parameter values

### Iteration 3: Using Lambda for Inline ID Generation

For simple transformations, you can use a lambda:

```python
# test_password_validator_ids.py
import pytest
from password_validator import validate_password

@pytest.mark.parametrize(
    "password,expected_valid,expected_message",
    [
        ("Short1!", False, "Password must be at least 8 characters"),
        ("password123!", False, "Password must contain at least one uppercase letter"),
        ("Password!", False, "Password must contain at least one number"),
        ("Password123", False, "Password must contain at least one special character"),
        ("Password123!", True, "Password is valid"),
    ],
    ids=lambda password, expected_valid, expected_message: (
        f"valid_{password}" if expected_valid else f"invalid_{password}"
    )
)
def test_password_validation(password, expected_valid, expected_message):
    is_valid, message = validate_password(password)
    assert is_valid == expected_valid
    assert message == expected_message
```

```bash
pytest test_password_validator_ids.py -v
```

**Output**:

```text
test_password_validator_ids.py::test_password_validation[invalid_Short1!] PASSED
test_password_validator_ids.py::test_password_validation[invalid_password123!] PASSED
test_password_validator_ids.py::test_password_validation[invalid_Password!] PASSED
test_password_validator_ids.py::test_password_validation[invalid_Password123] PASSED
test_password_validator_ids.py::test_password_validation[valid_Password123!] PASSED
```

### Iteration 4: Using `pytest.param` for Per-Case IDs

Sometimes you want to specify IDs for only **some** test cases. Use `pytest.param`:

```python
# test_password_validator_param_ids.py
import pytest
from password_validator import validate_password

@pytest.mark.parametrize(
    "password,expected_valid,expected_message",
    [
        pytest.param(
            "Short1!", False, "Password must be at least 8 characters",
            id="too_short"
        ),
        pytest.param(
            "password123!", False, "Password must contain at least one uppercase letter",
            id="no_uppercase"
        ),
        pytest.param(
            "Password!", False, "Password must contain at least one number",
            id="no_number"
        ),
        pytest.param(
            "Password123", False, "Password must contain at least one special character",
            id="no_special_char"
        ),
        pytest.param(
            "Password123!", True, "Password is valid",
            id="valid_password"
        ),
    ]
)
def test_password_validation(password, expected_valid, expected_message):
    is_valid, message = validate_password(password)
    assert is_valid == expected_valid
    assert message == expected_message
```

```bash
pytest test_password_validator_param_ids.py -v
```

**Output**:

```text
test_password_validator_param_ids.py::test_password_validation[too_short] PASSED
test_password_validator_param_ids.py::test_password_validation[no_uppercase] PASSED
test_password_validator_param_ids.py::test_password_validation[no_number] PASSED
test_password_validator_param_ids.py::test_password_validation[no_special_char] PASSED
test_password_validator_param_ids.py::test_password_validation[valid_password] PASSED
```

**Perfect!** Now our test names are:
- ✅ Descriptive
- ✅ Scannable
- ✅ Meaningful in CI logs

### Combining pytest.param with Marks

`pytest.param` can also carry **marks** (like `skip` or `xfail`):

```python
# test_password_validator_param_marks.py
import pytest
from password_validator import validate_password

@pytest.mark.parametrize(
    "password,expected_valid,expected_message",
    [
        pytest.param(
            "Short1!", False, "Password must be at least 8 characters",
            id="too_short"
        ),
        pytest.param(
            "password123!", False, "Password must contain at least one uppercase letter",
            id="no_uppercase"
        ),
        pytest.param(
            "Password!", False, "Password must contain at least one number",
            id="no_number"
        ),
        pytest.param(
            "Password123", False, "Password must contain at least one special character",
            id="no_special_char"
        ),
        pytest.param(
            "Password123!", True, "Password is valid",
            id="valid_password"
        ),
        pytest.param(
            "P@ssw0rd!", True, "Password is valid",
            id="unicode_special_char",
            marks=pytest.mark.xfail(reason="Unicode special chars not yet supported")
        ),
    ]
)
def test_password_validation(password, expected_valid, expected_message):
    is_valid, message = validate_password(password)
    assert is_valid == expected_valid
    assert message == expected_message
```

```bash
pytest test_password_validator_param_marks.py -v
```

**Output**:

```text
test_password_validator_param_marks.py::test_password_validation[too_short] PASSED
test_password_validator_param_marks.py::test_password_validation[no_uppercase] PASSED
test_password_validator_param_marks.py::test_password_validation[no_number] PASSED
test_password_validator_param_marks.py::test_password_validation[no_special_char] PASSED
test_password_validator_param_marks.py::test_password_validation[valid_password] PASSED
test_password_validator_param_marks.py::test_password_validation[unicode_special_char] XFAIL

==================== 5 passed, 1 xfailed in 0.03s ====================
```

The `unicode_special_char` test is marked as **expected to fail** (XFAIL), and pytest reports it separately.

### Demonstrating Failure with Custom IDs

Let's introduce a bug and see how custom IDs help with debugging:

```python
# password_validator_buggy.py
def validate_password(password: str) -> tuple[bool, str]:
    """Validate a password (with a bug)."""
    if len(password) < 8:
        return False, "Password must be at least 8 characters"
    
    if not any(c.isupper() for c in password):
        return False, "Password must contain at least one uppercase letter"
    
    # BUG: Forgot to check for numbers!
    # if not any(c.isdigit() for c in password):
    #     return False, "Password must contain at least one number"
    
    if not any(c in "!@#$%^&*()_+-=[]{}|;:,.<>?" for c in password):
        return False, "Password must contain at least one special character"
    
    return True, "Password is valid"
```

```bash
pytest test_password_validator_param_ids.py -v
```

**Output**:

```text
test_password_validator_param_ids.py::test_password_validation[too_short] PASSED
test_password_validator_param_ids.py::test_password_validation[no_uppercase] PASSED
test_password_validator_param_ids.py::test_password_validation[no_number] FAILED
test_password_validator_param_ids.py::test_password_validation[no_special_char] PASSED
test_password_validator_param_ids.py::test_password_validation[valid_password] PASSED

================================ FAILURES =================================
____________ test_password_validation[no_number] ____________

password = 'Password!', expected_valid = False
expected_message = 'Password must contain at least one number'

    @pytest.mark.parametrize(
        "password,expected_valid,expected_message",
        [
            pytest.param(
                "Short1!", False, "Password must be at least 8 characters",
                id="too_short"
            ),
            pytest.param(
                "password123!", False, "Password must contain at least one uppercase letter",
                id="no_uppercase"
            ),
            pytest.param(
                "Password!", False, "Password must contain at least one number",
                id="no_number"
            ),
            # ...
        ]
    )
    def test_password_validation(password, expected_valid, expected_message):
        is_valid, message = validate_password(password)
        assert is_valid == expected_valid
>       assert message == expected_message
E       AssertionError: assert 'Password is valid' == 'Password must...ast one number'
E         - Password must contain at least one number
E         + Password is valid

test_password_validator_param_ids.py:XX: AssertionError
==================== 1 failed, 4 passed in 0.03s ====================
```

### Diagnostic Analysis: The Value of Custom IDs

**The summary line**:
```
test_password_validation[no_number] FAILED
```

**Immediately tells us**:
- Which validation rule failed: `no_number`
- Without custom IDs, this would be: `test_password_validation[Password!-False-Password must contain at least one number]`

**In CI logs**, custom IDs make failures instantly recognizable:
```
❌ test_password_validation[no_number]
✅ test_password_validation[too_short]
✅ test_password_validation[no_uppercase]
```

vs. without custom IDs:
```
❌ test_password_validation[Password!-False-Password must contain at least one number]
✅ test_password_validation[Short1!-False-Password must be at least 8 characters]
✅ test_password_validation[password123!-False-Password must contain at least one uppercase letter]
```

### Best Practices for Test IDs

**DO**:
- ✅ Use descriptive names that explain **what** is being tested
- ✅ Keep IDs short but meaningful (aim for < 40 characters)
- ✅ Use consistent naming conventions across your test suite
- ✅ Include the key distinguishing feature of each test case

**DON'T**:
- ❌ Include the full parameter values in the ID (redundant)
- ❌ Use generic names like `test1`, `test2`, `case_a`
- ❌ Make IDs so long they wrap in terminal output
- ❌ Use special characters that might cause issues in file systems

### ID Generation Strategies

| Strategy | When to Use | Example |
|----------|-------------|---------|
| Manual list | Few test cases, very specific names | `ids=["valid", "invalid", "edge_case"]` |
| Function | Many test cases, algorithmic naming | `ids=lambda x, y: f"{x}_{y}"` |
| Lambda | Simple transformation | `ids=lambda x: f"test_{x}"` |
| `pytest.param` | Per-case control, need marks | `pytest.param(..., id="name", marks=...)` |

### The Complete Journey: From Repetition to Clarity

Let's review how far we've come with our password validator:

**Iteration 0: Separate test functions** (25 lines)

```python
def test_password_too_short():
    is_valid, message = validate_password("Short1!")
    assert not is_valid
    assert message == "Password must be at least 8 characters"

def test_password_no_uppercase():
    is_valid, message = validate_password("password123!")
    assert not is_valid
    assert message == "Password must contain at least one uppercase letter"

# ... 3 more functions
```

**Iteration 1: Basic parametrization** (13 lines)

```python
@pytest.mark.parametrize("password,expected_valid,expected_message", [
    ("Short1!", False, "Password must be at least 8 characters"),
    ("password123!", False, "Password must contain at least one uppercase letter"),
    ("Password!", False, "Password must contain at least one number"),
    ("Password123", False, "Password must contain at least one special character"),
    ("Password123!", True, "Password is valid"),
])
def test_password_validation(password, expected_valid, expected_message):
    is_valid, message = validate_password(password)
    assert is_valid == expected_valid
    assert message == expected_message
```

**Iteration 2: Custom IDs with pytest.param** (20 lines, but much clearer)

```python
@pytest.mark.parametrize("password,expected_valid,expected_message", [
    pytest.param("Short1!", False, "Password must be at least 8 characters", id="too_short"),
    pytest.param("password123!", False, "Password must contain at least one uppercase letter", id="no_uppercase"),
    pytest.param("Password!", False, "Password must contain at least one number", id="no_number"),
    pytest.param("Password123", False, "Password must contain at least one special character", id="no_special_char"),
    pytest.param("Password123!", True, "Password is valid", id="valid_password"),
])
def test_password_validation(password, expected_valid, expected_message):
    is_valid, message = validate_password(password)
    assert is_valid == expected_valid
    assert message == expected_message
```

**Benefits achieved**:
- ✅ **DRY**: Test logic defined once
- ✅ **Maintainable**: Change logic in one place
- ✅ **Extensible**: Add test case = add one line
- ✅ **Readable**: Custom IDs make failures obvious
- ✅ **Debuggable**: Clear test names in CI logs

### Lessons Learned

**Parametrization is about more than reducing code**. It's about:

1. **Expressing intent**: Test data as a table makes patterns visible
2. **Enabling growth**: Adding test cases becomes trivial
3. **Improving debugging**: Custom IDs make failures instantly recognizable
4. **Maintaining quality**: Change test logic once, all cases benefit

**The progression**:
1. Start with direct parametrization for simple cases
2. Add fixture parametrization when setup is complex
3. Use indirect parametrization when parameters need transformation
4. Apply custom IDs when test names matter (always in production code)

**The decision tree**:
```
Need to test multiple scenarios?
├─ Simple values, no setup needed
│  └─ Use @pytest.mark.parametrize (direct)
├─ Complex setup/teardown required
│  └─ Use fixture parametrization
├─ Parameters need transformation
│  └─ Use indirect parametrization
└─ Test names are unclear
   └─ Add custom IDs (function, lambda, or pytest.param)
```

You now have the complete toolkit for parametrized testing in pytest. Use these techniques to write comprehensive, maintainable test suites that scale with your codebase.
