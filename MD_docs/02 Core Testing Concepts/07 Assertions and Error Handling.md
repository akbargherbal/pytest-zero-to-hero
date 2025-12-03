# Chapter 7: Assertions and Error Handling

## Beyond Simple Assertions

## The Foundation: What We're Building On

In previous chapters, you've written tests using simple assertions like `assert result == expected`. These work beautifully for straightforward comparisons, but real-world testing demands more sophisticated assertion strategies. This chapter transforms your understanding of assertions from basic equality checks into a comprehensive toolkit for validating complex behaviors, handling errors gracefully, and writing tests that communicate clearly when they fail.

We'll build our understanding around a realistic scenario: testing a user registration system. This will be our **anchor example** that we'll refine through multiple iterations, each time encountering new challenges that demand more sophisticated assertion techniques.

## Phase 1: The Reference Implementation

Let's start with a user registration system that validates email addresses and passwords:

```python
# user_registration.py
import re
from typing import Optional

class RegistrationError(Exception):
    """Base exception for registration failures"""
    pass

class InvalidEmailError(RegistrationError):
    """Raised when email format is invalid"""
    pass

class WeakPasswordError(RegistrationError):
    """Raised when password doesn't meet requirements"""
    pass

class User:
    def __init__(self, email: str, username: str):
        self.email = email
        self.username = username
        self.is_active = False
    
    def __repr__(self):
        return f"User(email='{self.email}', username='{self.username}')"

def validate_email(email: str) -> bool:
    """Check if email has valid format"""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return bool(re.match(pattern, email))

def validate_password(password: str) -> tuple[bool, Optional[str]]:
    """
    Validate password strength.
    Returns (is_valid, error_message)
    """
    if len(password) < 8:
        return False, "Password must be at least 8 characters"
    if not any(c.isupper() for c in password):
        return False, "Password must contain at least one uppercase letter"
    if not any(c.isdigit() for c in password):
        return False, "Password must contain at least one digit"
    return True, None

def register_user(email: str, username: str, password: str) -> User:
    """
    Register a new user with validation.
    Raises appropriate exceptions on validation failure.
    """
    if not validate_email(email):
        raise InvalidEmailError(f"Invalid email format: {email}")
    
    is_valid, error_msg = validate_password(password)
    if not is_valid:
        raise WeakPasswordError(error_msg)
    
    user = User(email=email, username=username)
    # In real system, would save to database here
    return user
```

Now let's write our initial test suite using simple assertions:

```python
# test_registration_v1.py
import pytest
from user_registration import (
    register_user, 
    User,
    InvalidEmailError,
    WeakPasswordError
)

def test_successful_registration():
    """Test that valid credentials create a user"""
    result = register_user(
        email="alice@example.com",
        username="alice",
        password="SecurePass123"
    )
    
    # Simple assertions
    assert result.email == "alice@example.com"
    assert result.username == "alice"
    assert result.is_active == False

def test_invalid_email_rejected():
    """Test that invalid email raises exception"""
    # This will fail - we need better assertion techniques
    register_user(
        email="not-an-email",
        username="bob",
        password="SecurePass123"
    )

def test_weak_password_rejected():
    """Test that weak password raises exception"""
    # This will also fail
    register_user(
        email="charlie@example.com",
        username="charlie",
        password="weak"
    )
```

Let's run these tests to see what happens:

```bash
pytest test_registration_v1.py -v
```

**Output**:
```
======================== test session starts =========================
collected 3 items

test_registration_v1.py::test_successful_registration PASSED    [ 33%]
test_registration_v1.py::test_invalid_email_rejected FAILED     [ 66%]
test_registration_v1.py::test_weak_password_rejected FAILED     [100%]

============================== FAILURES ==============================
_________________ test_invalid_email_rejected ________________________

    def test_invalid_email_rejected():
        """Test that invalid email raises exception"""
>       register_user(
            email="not-an-email",
            username="bob",
            password="SecurePass123"
        )
E       user_registration.InvalidEmailError: Invalid email format: not-an-email

test_registration_v1.py:24: InvalidEmailError
_________________ test_weak_password_rejected ________________________

    def test_weak_password_rejected():
        """Test that weak password raises exception"""
>       register_user(
            email="charlie@example.com",
            username="charlie",
            password="weak"
        )
E       user_registration.WeakPasswordError: Password must be at least 8 characters

test_registration_v1.py:32: WeakPasswordError
==================== 2 failed, 1 passed in 0.12s =====================
```

### Diagnostic Analysis: Reading the Failure

**The complete output shows**:

1. **The summary line**: `test_invalid_email_rejected FAILED`
   - What this tells us: The test itself failed, not just an assertion

2. **The traceback**:
```
>       register_user(
            email="not-an-email",
            username="bob",
            password="SecurePass123"
        )
E       user_registration.InvalidEmailError: Invalid email format: not-an-email
```
   - What this tells us: An exception was raised but not caught
   - Key line: The `E` marker shows the exception propagated out of the test

3. **No assertion introspection**:
   - What this tells us: We never reached an assertion—the exception killed the test first

**Root cause identified**: We're testing that exceptions *should* be raised, but we have no mechanism to catch and verify them.

**Why the current approach can't solve this**: Simple assertions like `assert result == expected` only work when code completes successfully. When we *expect* an exception, we need a way to catch it, verify it's the right type, and optionally inspect its message.

**What we need**: A context manager that captures exceptions so we can assert on their properties.

## Iteration 1: Testing Expected Exceptions

Our tests are failing because exceptions are propagating uncaught. We need `pytest.raises()` to capture and verify exceptions:

```python
# test_registration_v2.py
import pytest
from user_registration import (
    register_user,
    User,
    InvalidEmailError,
    WeakPasswordError
)

def test_successful_registration():
    """Test that valid credentials create a user"""
    result = register_user(
        email="alice@example.com",
        username="alice",
        password="SecurePass123"
    )
    
    assert result.email == "alice@example.com"
    assert result.username == "alice"
    assert result.is_active == False

def test_invalid_email_rejected():
    """Test that invalid email raises InvalidEmailError"""
    with pytest.raises(InvalidEmailError):
        register_user(
            email="not-an-email",
            username="bob",
            password="SecurePass123"
        )

def test_weak_password_rejected():
    """Test that weak password raises WeakPasswordError"""
    with pytest.raises(WeakPasswordError):
        register_user(
            email="charlie@example.com",
            username="charlie",
            password="weak"
        )
```

Run the updated tests:

```bash
pytest test_registration_v2.py -v
```

**Output**:
```
======================== test session starts =========================
collected 3 items

test_registration_v2.py::test_successful_registration PASSED    [ 33%]
test_registration_v2.py::test_invalid_email_rejected PASSED     [ 66%]
test_registration_v2.py::test_weak_password_rejected PASSED     [100%]

========================= 3 passed in 0.08s ==========================
```

**Expected vs. Actual improvement**: All tests now pass. The `pytest.raises()` context manager captures the expected exceptions, preventing them from failing the test.

### How pytest.raises() Works

The `pytest.raises()` context manager:

1. **Enters a protected block**: Code inside the `with` block is monitored
2. **Expects an exception**: If the specified exception type is raised, it's caught
3. **Validates the exception type**: Only the exact type (or subclasses) pass
4. **Fails if no exception occurs**: If the block completes without raising, the test fails

Think of it as an "assertion that an exception happens."

**Current limitation**: We're only verifying that *some* exception of the right type was raised. We're not checking the error message, which means we can't distinguish between different failure scenarios. What if we want to verify that the password error message specifically mentions "8 characters" vs "uppercase letter"?

## Iteration 2: Inspecting Exception Messages

Let's add a test that needs to verify the specific error message:

```python
# test_registration_v3.py
import pytest
from user_registration import (
    register_user,
    InvalidEmailError,
    WeakPasswordError
)

def test_password_too_short_specific_message():
    """Test that short password gives specific error message"""
    with pytest.raises(WeakPasswordError):
        register_user(
            email="dave@example.com",
            username="dave",
            password="Short1"  # Only 6 characters
        )
    # How do we check the message says "at least 8 characters"?

def test_password_missing_uppercase_specific_message():
    """Test that password without uppercase gives specific error"""
    with pytest.raises(WeakPasswordError):
        register_user(
            email="eve@example.com",
            username="eve",
            password="lowercase123"  # No uppercase
        )
    # How do we check the message says "uppercase letter"?
```

Run these tests:

```bash
pytest test_registration_v3.py::test_password_too_short_specific_message -v
pytest test_registration_v3.py::test_password_missing_uppercase_specific_message -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_registration_v3.py::test_password_too_short_specific_message PASSED [100%]

========================= 1 passed in 0.05s ==========================

======================== test session starts =========================
collected 1 item

test_registration_v3.py::test_password_missing_uppercase_specific_message PASSED [100%]

========================= 1 passed in 0.05s ==========================
```

### Diagnostic Analysis: The Hidden Problem

**The tests pass, but are they testing enough?**

Both tests pass because they verify that `WeakPasswordError` is raised. But they don't verify *which* validation rule failed. This is a **false positive**—the tests would still pass even if we swapped the error messages.

Let's prove this by introducing a bug:

```python
# user_registration_buggy.py (modified validate_password)
def validate_password(password: str) -> tuple[bool, Optional[str]]:
    """Validate password strength - WITH BUG"""
    if len(password) < 8:
        return False, "WRONG MESSAGE"  # Bug: wrong error message
    if not any(c.isupper() for c in password):
        return False, "ALSO WRONG"  # Bug: wrong error message
    if not any(c.isdigit() for c in password):
        return False, "Password must contain at least one digit"
    return True, None
```

Run the tests again with the buggy version:

```bash
pytest test_registration_v3.py -v
```

**Output**:
```
======================== test session starts =========================
collected 2 items

test_registration_v3.py::test_password_too_short_specific_message PASSED [50%]
test_registration_v3.py::test_password_missing_uppercase_specific_message PASSED [100%]

========================= 2 passed in 0.06s ==========================
```

**Root cause identified**: Our tests pass even with wrong error messages because we're not inspecting the exception content.

**What we need**: Access to the exception object so we can assert on its message.

### Solution: Capturing the Exception Object

`pytest.raises()` can capture the exception into a variable using the `as` keyword:

```python
# test_registration_v4.py
import pytest
from user_registration import (
    register_user,
    WeakPasswordError
)

def test_password_too_short_specific_message():
    """Test that short password gives specific error message"""
    with pytest.raises(WeakPasswordError) as exc_info:
        register_user(
            email="dave@example.com",
            username="dave",
            password="Short1"
        )
    
    # Now we can inspect the exception
    assert "at least 8 characters" in str(exc_info.value)

def test_password_missing_uppercase_specific_message():
    """Test that password without uppercase gives specific error"""
    with pytest.raises(WeakPasswordError) as exc_info:
        register_user(
            email="eve@example.com",
            username="eve",
            password="lowercase123"
        )
    
    assert "uppercase letter" in str(exc_info.value)

def test_password_missing_digit_specific_message():
    """Test that password without digit gives specific error"""
    with pytest.raises(WeakPasswordError) as exc_info:
        register_user(
            email="frank@example.com",
            username="frank",
            password="NoDigitsHere"
        )
    
    assert "at least one digit" in str(exc_info.value)
```

Run with the correct implementation:

```bash
pytest test_registration_v4.py -v
```

**Output**:
```
======================== test session starts =========================
collected 3 items

test_registration_v4.py::test_password_too_short_specific_message PASSED [33%]
test_registration_v4.py::test_password_missing_uppercase_specific_message PASSED [66%]
test_registration_v4.py::test_password_missing_digit_specific_message PASSED [100%]

========================= 3 passed in 0.07s ==========================
```

Now run with the buggy implementation:

```bash
pytest test_registration_v4.py -v  # Using buggy version
```

**Output**:
```
======================== test session starts =========================
collected 3 items

test_registration_v4.py::test_password_too_short_specific_message FAILED [33%]
test_registration_v4.py::test_password_missing_uppercase_specific_message FAILED [66%]
test_registration_v4.py::test_password_missing_digit_specific_message PASSED [100%]

============================== FAILURES ==============================
_____________ test_password_too_short_specific_message _______________

    def test_password_too_short_specific_message():
        with pytest.raises(WeakPasswordError) as exc_info:
            register_user(
                email="dave@example.com",
                username="dave",
                password="Short1"
            )
        
>       assert "at least 8 characters" in str(exc_info.value)
E       AssertionError: assert 'at least 8 characters' in 'WRONG MESSAGE'
E        +  where 'WRONG MESSAGE' = str(WeakPasswordError('WRONG MESSAGE'))

test_registration_v4.py:16: AssertionError
____________ test_password_missing_uppercase_specific_message ________

    def test_password_missing_uppercase_specific_message():
        with pytest.raises(WeakPasswordError) as exc_info:
            register_user(
                email="eve@example.com",
                username="eve",
                password="lowercase123"
            )
        
>       assert "uppercase letter" in str(exc_info.value)
E       AssertionError: assert 'uppercase letter' in 'ALSO WRONG'
E        +  where 'ALSO WRONG' = str(WeakPasswordError('ALSO WRONG'))

test_registration_v4.py:27: AssertionError
==================== 2 failed, 1 passed in 0.11s =====================
```

**Expected vs. Actual improvement**: Now our tests correctly fail when error messages are wrong. The `exc_info.value` gives us access to the actual exception object, and we can assert on its string representation.

### Understanding ExceptionInfo

When you use `with pytest.raises(ExceptionType) as exc_info:`, the `exc_info` object has several useful attributes:

- `exc_info.value`: The actual exception instance
- `exc_info.type`: The exception class
- `exc_info.traceback`: The traceback object

Most commonly, you'll use `exc_info.value` to access the exception message.

**Current limitation**: We're using substring matching (`in str(exc_info.value)`), which is flexible but imprecise. What if we want exact message matching? Or regex patterns? Or to verify exception attributes beyond the message?

## Assertion Introspection: Reading Failure Messages

## The Power of Pytest's Assertion Rewriting

Before we continue refining our exception testing, let's understand one of pytest's most powerful features: **assertion introspection**. This is the magic that makes pytest's failure messages so informative.

When you write `assert x == y`, pytest doesn't just evaluate the boolean result. It rewrites your assertion at import time to capture intermediate values, allowing it to show you *exactly* what went wrong.

## Iteration 3: Complex Object Comparisons

Let's extend our user registration system to return more complex data:

```python
# user_registration_extended.py
from dataclasses import dataclass
from typing import List, Optional
from datetime import datetime

@dataclass
class UserProfile:
    email: str
    username: str
    full_name: str
    created_at: datetime
    roles: List[str]
    is_active: bool
    metadata: dict

def create_user_profile(
    email: str,
    username: str,
    full_name: str,
    roles: Optional[List[str]] = None
) -> UserProfile:
    """Create a complete user profile"""
    return UserProfile(
        email=email,
        username=username,
        full_name=full_name,
        created_at=datetime.now(),
        roles=roles or ["user"],
        is_active=False,
        metadata={"source": "registration", "version": "1.0"}
    )
```

Now let's write a test with a deliberate mistake to see assertion introspection in action:

```python
# test_profile_v1.py
from user_registration_extended import create_user_profile

def test_user_profile_creation():
    """Test that user profile is created with correct attributes"""
    profile = create_user_profile(
        email="alice@example.com",
        username="alice",
        full_name="Alice Anderson"
    )
    
    # Deliberate mistake: wrong username
    expected_username = "alicia"  # Bug: should be "alice"
    assert profile.username == expected_username
```

Run the test:

```bash
pytest test_profile_v1.py -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_profile_v1.py::test_user_profile_creation FAILED           [100%]

============================== FAILURES ==============================
_________________ test_user_profile_creation _________________________

    def test_user_profile_creation():
        profile = create_user_profile(
            email="alice@example.com",
            username="alice",
            full_name="Alice Anderson"
        )
        
        expected_username = "alicia"
>       assert profile.username == expected_username
E       AssertionError: assert 'alice' == 'alicia'
E         - alicia
E         + alice

test_profile_v1.py:13: AssertionError
========================= 1 failed in 0.09s ==========================
```

### Diagnostic Analysis: Understanding Introspection

**The assertion introspection shows**:

1. **The assertion line**: `assert profile.username == expected_username`
   - What this tells us: The exact comparison that failed

2. **The evaluated values**: `assert 'alice' == 'alicia'`
   - What this tells us: Pytest evaluated both sides and shows the actual values
   - Key insight: We see both `profile.username` (evaluated to `'alice'`) and `expected_username` (evaluated to `'alicia'`)

3. **The diff visualization**:
```
E       - alicia
E       + alice
```
   - What this tells us: The `-` line shows what was expected, `+` shows what was actual
   - This is a unified diff format, familiar from version control

**Root cause identified**: The expected value is wrong in our test.

**Why this matters**: Without introspection, we'd only see `AssertionError` with no context. Pytest's rewriting gives us the full picture.

## Comparing Complex Objects

Let's test a more complex scenario where we compare entire objects:

```python
# test_profile_v2.py
from user_registration_extended import create_user_profile, UserProfile
from datetime import datetime

def test_user_profile_complete_comparison():
    """Test complete profile structure"""
    profile = create_user_profile(
        email="bob@example.com",
        username="bob",
        full_name="Bob Builder",
        roles=["user", "admin"]
    )
    
    # Create expected profile with deliberate mistakes
    expected = UserProfile(
        email="bob@example.com",
        username="bob",
        full_name="Bob Builder",
        created_at=datetime(2024, 1, 1, 12, 0, 0),  # Wrong: won't match actual
        roles=["user", "moderator"],  # Wrong: should be "admin"
        is_active=True,  # Wrong: should be False
        metadata={"source": "registration", "version": "1.0"}
    )
    
    assert profile == expected
```

Run the test:

```bash
pytest test_profile_v2.py -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_profile_v2.py::test_user_profile_complete_comparison FAILED [100%]

============================== FAILURES ==============================
__________ test_user_profile_complete_comparison _____________________

    def test_user_profile_complete_comparison():
        profile = create_user_profile(
            email="bob@example.com",
            username="bob",
            full_name="Bob Builder",
            roles=["user", "admin"]
        )
        
        expected = UserProfile(
            email="bob@example.com",
            username="bob",
            full_name="Bob Builder",
            created_at=datetime(2024, 1, 1, 12, 0, 0),
            roles=["user", "moderator"],
            is_active=True,
            metadata={"source": "registration", "version": "1.0"}
        )
        
>       assert profile == expected
E       AssertionError: assert UserProfile(em...version': '1.0'}) == UserProfile(em...version': '1.0'})
E         
E         Omitting 3 identical items, use -vv to show
E         Differing attributes:
E         ['created_at', 'is_active', 'roles']

test_profile_v2.py:24: AssertionError
========================= 1 failed in 0.10s ==========================
```

### Diagnostic Analysis: Dataclass Comparison

**The introspection shows**:

1. **Omitted identical items**: Pytest recognizes that some fields match and focuses on differences
2. **Differing attributes**: Lists exactly which fields don't match: `created_at`, `is_active`, `roles`

But we want more detail. Let's use pytest's verbose mode:

```bash
pytest test_profile_v2.py -vv
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_profile_v2.py::test_user_profile_complete_comparison FAILED [100%]

============================== FAILURES ==============================
__________ test_user_profile_complete_comparison _____________________

    def test_user_profile_complete_comparison():
        profile = create_user_profile(
            email="bob@example.com",
            username="bob",
            full_name="Bob Builder",
            roles=["user", "admin"]
        )
        
        expected = UserProfile(
            email="bob@example.com",
            username="bob",
            full_name="Bob Builder",
            created_at=datetime(2024, 1, 1, 12, 0, 0),
            roles=["user", "moderator"],
            is_active=True,
            metadata={"source": "registration", "version": "1.0"}
        )
        
>       assert profile == expected
E       AssertionError: assert UserProfile(email='bob@example.com', username='bob', full_name='Bob Builder', created_at=datetime.datetime(2024, 12, 19, 10, 30, 45, 123456), roles=['user', 'admin'], is_active=False, metadata={'source': 'registration', 'version': '1.0'}) == UserProfile(email='bob@example.com', username='bob', full_name='Bob Builder', created_at=datetime.datetime(2024, 1, 1, 12, 0, 0), roles=['user', 'moderator'], is_active=True, metadata={'source': 'registration', 'version': '1.0'})
E         
E         Matching attributes:
E         ['email', 'full_name', 'metadata', 'username']
E         Differing attributes:
E         ['created_at', 'is_active', 'roles']
E         
E         Drill down into differing attribute created_at:
E           created_at: datetime.datetime(2024, 12, 19, 10, 30, 45, 123456) != datetime.datetime(2024, 1, 1, 12, 0, 0)
E         
E         Drill down into differing attribute is_active:
E           is_active: False != True
E         
E         Drill down into differing attribute roles:
E           roles: ['user', 'admin'] != ['user', 'moderator']

test_profile_v2.py:24: AssertionError
========================= 1 failed in 0.11s ==========================
```

**Expected vs. Actual improvement**: With `-vv`, pytest shows:
- Full object representations
- Which attributes match
- Which attributes differ
- Detailed drill-down into each differing attribute

This level of detail makes debugging complex object comparisons trivial.

## Iteration 4: Collection Comparisons

Let's test list and dictionary comparisons to see how introspection handles collections:

```python
# test_collections.py
def test_list_comparison():
    """Test list comparison with differences"""
    actual_roles = ["user", "admin", "moderator"]
    expected_roles = ["user", "editor", "moderator"]
    
    assert actual_roles == expected_roles

def test_dict_comparison():
    """Test dictionary comparison with differences"""
    actual_metadata = {
        "source": "registration",
        "version": "1.0",
        "features": ["email", "2fa"],
        "limits": {"api_calls": 1000}
    }
    
    expected_metadata = {
        "source": "registration",
        "version": "2.0",  # Different
        "features": ["email", "sso"],  # Different
        "limits": {"api_calls": 1000}
    }
    
    assert actual_metadata == expected_metadata
```

Run the tests:

```bash
pytest test_collections.py -vv
```

**Output**:
```
======================== test session starts =========================
collected 2 items

test_collections.py::test_list_comparison FAILED                [50%]
test_collections.py::test_dict_comparison FAILED                [100%]

============================== FAILURES ==============================
______________________ test_list_comparison __________________________

    def test_list_comparison():
        actual_roles = ["user", "admin", "moderator"]
        expected_roles = ["user", "editor", "moderator"]
        
>       assert actual_roles == expected_roles
E       AssertionError: assert ['user', 'admin', 'moderator'] == ['user', 'editor', 'moderator']
E         At index 1 diff: 'admin' != 'editor'
E         Use -v to get more diff

test_collections.py:5: AssertionError
______________________ test_dict_comparison __________________________

    def test_dict_comparison():
        actual_metadata = {
            "source": "registration",
            "version": "1.0",
            "features": ["email", "2fa"],
            "limits": {"api_calls": 1000}
        }
        
        expected_metadata = {
            "source": "registration",
            "version": "2.0",
            "features": ["email", "sso"],
            "limits": {"api_calls": 1000}
        }
        
>       assert actual_metadata == expected_metadata
E       AssertionError: assert {'features': [...], ...} == {'features': [...], ...}
E         
E         Omitting 2 identical items, use -vv to show
E         Differing items:
E         {'version': '1.0'} != {'version': '2.0'}
E         {'features': ['email', '2fa']} != {'features': ['email', 'sso']}

test_collections.py:20: AssertionError
========================= 1 failed, 1 passed in 0.12s =====================
```

### Diagnostic Analysis: Collection Introspection

**For lists**:
- Pytest identifies the exact index where differences occur: `At index 1 diff: 'admin' != 'editor'`
- This pinpoints the problem immediately

**For dictionaries**:
- Pytest shows which keys have matching values (omitted for brevity)
- Pytest shows which keys have differing values with both sides of the comparison
- Nested structures are handled intelligently

**Key insight**: Pytest's introspection adapts to the data structure being compared, providing context-appropriate failure messages.

## When Introspection Isn't Enough

Sometimes you need more control over comparison logic. Let's see a case where simple equality fails:

```python
# test_floating_point.py
def test_calculation_result():
    """Test a calculation that produces floating point results"""
    result = 0.1 + 0.2
    expected = 0.3
    
    assert result == expected
```

Run the test:

```bash
pytest test_floating_point.py -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_floating_point.py::test_calculation_result FAILED          [100%]

============================== FAILURES ==============================
__________________ test_calculation_result ___________________________

    def test_calculation_result():
        result = 0.1 + 0.2
        expected = 0.3
        
>       assert result == expected
E       AssertionError: assert 0.30000000000000004 == 0.3

test_floating_point.py:5: AssertionError
========================= 1 failed in 0.06s ==========================
```

### Diagnostic Analysis: Floating Point Precision

**The introspection reveals**:
- `0.1 + 0.2` evaluates to `0.30000000000000004`, not `0.3`
- This is a fundamental limitation of binary floating point representation

**Root cause identified**: We need approximate comparison, not exact equality.

**What we need**: A way to assert "close enough" rather than "exactly equal."

This is where we move beyond simple assertions to more sophisticated comparison strategies, which we'll cover in Section 7.3.

## Summary: Reading Pytest's Failure Messages

When a test fails, pytest provides a structured narrative:

1. **Test location**: Which file and function failed
2. **Assertion line**: The exact line of code that failed
3. **Evaluated values**: What each side of the comparison actually was
4. **Diff visualization**: How the values differ (for strings, lists, dicts, objects)
5. **Context**: Surrounding code for understanding the failure

**How to read a failure systematically**:

1. Start with the summary line to identify which test failed
2. Look at the assertion line to see what was being compared
3. Read the evaluated values to understand what actually happened
4. Study the diff to pinpoint exact differences
5. Trace back through the code to understand why the values differ

This systematic approach transforms debugging from guesswork into methodical problem-solving.

## Custom Assertion Messages

## When Default Messages Aren't Enough

Pytest's introspection is powerful, but sometimes you need to add domain-specific context to make failures immediately understandable. This is especially true when:

- The assertion logic is complex
- The failure could have multiple causes
- You want to guide the developer toward the fix
- The values being compared need interpretation

## Iteration 5: Adding Context to Assertions

Let's return to our user registration system and add a feature that requires custom messages:

```python
# user_validation.py
from typing import List, Tuple

def check_username_availability(
    username: str,
    existing_usernames: List[str]
) -> Tuple[bool, str]:
    """
    Check if username is available and meets requirements.
    Returns (is_available, reason)
    """
    if len(username) < 3:
        return False, "too_short"
    if len(username) > 20:
        return False, "too_long"
    if not username.isalnum():
        return False, "invalid_characters"
    if username.lower() in [u.lower() for u in existing_usernames]:
        return False, "already_taken"
    return True, "available"

def validate_user_registration(
    email: str,
    username: str,
    password: str,
    existing_usernames: List[str]
) -> Tuple[bool, List[str]]:
    """
    Validate complete registration.
    Returns (is_valid, list_of_errors)
    """
    errors = []
    
    # Email validation
    if "@" not in email or "." not in email.split("@")[-1]:
        errors.append("invalid_email")
    
    # Username validation
    is_available, reason = check_username_availability(username, existing_usernames)
    if not is_available:
        errors.append(f"username_{reason}")
    
    # Password validation
    if len(password) < 8:
        errors.append("password_too_short")
    if not any(c.isupper() for c in password):
        errors.append("password_no_uppercase")
    if not any(c.isdigit() for c in password):
        errors.append("password_no_digit")
    
    return len(errors) == 0, errors
```

Now let's write tests without custom messages first:

```python
# test_validation_v1.py
from user_validation import validate_user_registration

def test_valid_registration():
    """Test that valid data passes all checks"""
    is_valid, errors = validate_user_registration(
        email="alice@example.com",
        username="alice",
        password="SecurePass123",
        existing_usernames=["bob", "charlie"]
    )
    
    assert is_valid
    assert errors == []

def test_multiple_validation_errors():
    """Test that multiple errors are caught"""
    is_valid, errors = validate_user_registration(
        email="not-an-email",
        username="ab",  # Too short
        password="weak",  # Too short, no uppercase, no digit
        existing_usernames=[]
    )
    
    assert not is_valid
    # This assertion will fail - let's see the default message
    assert len(errors) == 3  # Wrong: should be 4
```

Run the test:

```bash
pytest test_validation_v1.py::test_multiple_validation_errors -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_validation_v1.py::test_multiple_validation_errors FAILED   [100%]

============================== FAILURES ==============================
_____________ test_multiple_validation_errors ________________________

    def test_multiple_validation_errors():
        is_valid, errors = validate_user_registration(
            email="not-an-email",
            username="ab",
            password="weak",
            existing_usernames=[]
        )
        
        assert not is_valid
>       assert len(errors) == 3
E       AssertionError: assert 5 == 3
E        +  where 5 = len(['invalid_email', 'username_too_short', 'password_too_short', 'password_no_uppercase', 'password_no_digit'])

test_validation_v1.py:20: AssertionError
========================= 1 failed in 0.08s ==========================
```

### Diagnostic Analysis: When Introspection Needs Help

**The introspection shows**:
- `len(errors) == 5`, not `3`
- The actual errors list: `['invalid_email', 'username_too_short', 'password_too_short', 'password_no_uppercase', 'password_no_digit']`

**The problem**: While we can see the error list, we have to mentally parse it to understand what went wrong. The assertion doesn't explain *why* we expected 3 errors or *which* errors we were expecting.

**What we need**: A custom message that explains the expectation and helps debug the failure.

## Adding Custom Messages with the Second Argument

Python's `assert` statement accepts an optional second argument—a message to display on failure:

```python
# test_validation_v2.py
from user_validation import validate_user_registration

def test_multiple_validation_errors_with_message():
    """Test that multiple errors are caught - with custom message"""
    is_valid, errors = validate_user_registration(
        email="not-an-email",
        username="ab",
        password="weak",
        existing_usernames=[]
    )
    
    assert not is_valid
    
    expected_errors = {
        "invalid_email",
        "username_too_short",
        "password_too_short"
    }
    actual_errors = set(errors)
    
    assert actual_errors == expected_errors, (
        f"Expected exactly these errors: {expected_errors}\n"
        f"But got: {actual_errors}\n"
        f"Missing: {expected_errors - actual_errors}\n"
        f"Extra: {actual_errors - expected_errors}"
    )
```

Run the test:

```bash
pytest test_validation_v2.py::test_multiple_validation_errors_with_message -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_validation_v2.py::test_multiple_validation_errors_with_message FAILED [100%]

============================== FAILURES ==============================
______ test_multiple_validation_errors_with_message __________________

    def test_multiple_validation_errors_with_message():
        is_valid, errors = validate_user_registration(
            email="not-an-email",
            username="ab",
            password="weak",
            existing_usernames=[]
        )
        
        assert not is_valid
        
        expected_errors = {
            "invalid_email",
            "username_too_short",
            "password_too_short"
        }
        actual_errors = set(errors)
        
>       assert actual_errors == expected_errors, (
            f"Expected exactly these errors: {expected_errors}\n"
            f"But got: {actual_errors}\n"
            f"Missing: {expected_errors - actual_errors}\n"
            f"Extra: {actual_errors - expected_errors}"
        )
E       AssertionError: Expected exactly these errors: {'username_too_short', 'password_too_short', 'invalid_email'}
E       But got: {'password_no_digit', 'username_too_short', 'password_too_short', 'password_no_uppercase', 'invalid_email'}
E       Missing: set()
E       Extra: {'password_no_digit', 'password_no_uppercase'}
E       
E       assert {'invalid_emai...o_uppercase'} == {'invalid_emai...sword_too_short'}
E         Extra items in the left set:
E         'password_no_digit'
E         'password_no_uppercase'

test_validation_v2.py:18: AssertionError
========================= 1 failed in 0.09s ==========================
```

**Expected vs. Actual improvement**: Now the failure message immediately tells us:
- What we expected
- What we got
- What's missing (none in this case)
- What's extra (`password_no_digit`, `password_no_uppercase`)

This makes the problem obvious: we forgot to account for the additional password validation rules.

## Iteration 6: Contextual Messages for Complex Logic

Let's test a more complex scenario where custom messages provide essential context:

```python
# user_permissions.py
from typing import List, Set
from enum import Enum

class Permission(Enum):
    READ = "read"
    WRITE = "write"
    DELETE = "delete"
    ADMIN = "admin"

class Role:
    def __init__(self, name: str, permissions: Set[Permission]):
        self.name = name
        self.permissions = permissions

def check_user_can_perform_action(
    user_roles: List[Role],
    required_permission: Permission,
    resource_owner: str,
    current_user: str
) -> Tuple[bool, str]:
    """
    Check if user can perform action.
    Returns (can_perform, reason)
    """
    # Collect all permissions from all roles
    user_permissions = set()
    for role in user_roles:
        user_permissions.update(role.permissions)
    
    # Admin can do anything
    if Permission.ADMIN in user_permissions:
        return True, "admin_override"
    
    # Owner can do anything to their own resources
    if resource_owner == current_user:
        return True, "owner_privilege"
    
    # Check if user has required permission
    if required_permission in user_permissions:
        return True, "has_permission"
    
    return False, "insufficient_permissions"
```

Now let's write tests with detailed custom messages:

```python
# test_permissions.py
import pytest
from user_permissions import (
    Permission,
    Role,
    check_user_can_perform_action
)

def test_admin_can_delete_any_resource():
    """Test that admin role grants delete permission on any resource"""
    admin_role = Role("admin", {Permission.ADMIN})
    
    can_perform, reason = check_user_can_perform_action(
        user_roles=[admin_role],
        required_permission=Permission.DELETE,
        resource_owner="alice",
        current_user="bob"
    )
    
    assert can_perform, (
        f"Admin should be able to delete any resource, but got: {reason}\n"
        f"User roles: {[r.name for r in [admin_role]]}\n"
        f"Required permission: {Permission.DELETE}\n"
        f"Resource owner: alice, Current user: bob"
    )
    
    assert reason == "admin_override", (
        f"Expected reason 'admin_override' for admin action, got: {reason}"
    )

def test_user_cannot_delete_others_resources():
    """Test that regular user cannot delete others' resources"""
    reader_role = Role("reader", {Permission.READ})
    
    can_perform, reason = check_user_can_perform_action(
        user_roles=[reader_role],
        required_permission=Permission.DELETE,
        resource_owner="alice",
        current_user="bob"
    )
    
    assert not can_perform, (
        f"User without delete permission should not be able to delete, "
        f"but got can_perform={can_perform}\n"
        f"User roles: {[r.name for r in [reader_role]]}\n"
        f"User permissions: {reader_role.permissions}\n"
        f"Required permission: {Permission.DELETE}\n"
        f"Resource owner: alice, Current user: bob"
    )

def test_owner_can_delete_own_resource():
    """Test that resource owner can delete their own resource"""
    reader_role = Role("reader", {Permission.READ})
    
    can_perform, reason = check_user_can_perform_action(
        user_roles=[reader_role],
        required_permission=Permission.DELETE,
        resource_owner="alice",
        current_user="alice"  # Same user
    )
    
    assert can_perform, (
        f"Resource owner should be able to delete their own resource\n"
        f"User roles: {[r.name for r in [reader_role]]}\n"
        f"User permissions: {reader_role.permissions}\n"
        f"Required permission: {Permission.DELETE}\n"
        f"Resource owner: alice, Current user: alice\n"
        f"Got reason: {reason}"
    )
    
    assert reason == "owner_privilege", (
        f"Expected reason 'owner_privilege' for owner action, got: {reason}"
    )
```

Run the tests:

```bash
pytest test_permissions.py -v
```

**Output**:
```
======================== test session starts =========================
collected 3 items

test_permissions.py::test_admin_can_delete_any_resource PASSED  [33%]
test_permissions.py::test_user_cannot_delete_others_resources PASSED [66%]
test_permissions.py::test_owner_can_delete_own_resource PASSED  [100%]

========================= 3 passed in 0.08s ==========================
```

Now let's introduce a bug to see the custom messages in action:

```python
# user_permissions_buggy.py (modified check function)
def check_user_can_perform_action(
    user_roles: List[Role],
    required_permission: Permission,
    resource_owner: str,
    current_user: str
) -> Tuple[bool, str]:
    """Check if user can perform action - WITH BUG"""
    user_permissions = set()
    for role in user_roles:
        user_permissions.update(role.permissions)
    
    # BUG: Removed admin override check
    
    # Owner can do anything to their own resources
    if resource_owner == current_user:
        return True, "owner_privilege"
    
    if required_permission in user_permissions:
        return True, "has_permission"
    
    return False, "insufficient_permissions"
```

Run the tests with the buggy version:

```bash
pytest test_permissions.py::test_admin_can_delete_any_resource -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_permissions.py::test_admin_can_delete_any_resource FAILED  [100%]

============================== FAILURES ==============================
____________ test_admin_can_delete_any_resource ______________________

    def test_admin_can_delete_any_resource():
        admin_role = Role("admin", {Permission.ADMIN})
        
        can_perform, reason = check_user_can_perform_action(
            user_roles=[admin_role],
            required_permission=Permission.DELETE,
            resource_owner="alice",
            current_user="bob"
        )
        
>       assert can_perform, (
            f"Admin should be able to delete any resource, but got: {reason}\n"
            f"User roles: {[r.name for r in [admin_role]]}\n"
            f"Required permission: {Permission.DELETE}\n"
            f"Resource owner: alice, Current user: bob"
        )
E       AssertionError: Admin should be able to delete any resource, but got: insufficient_permissions
E       User roles: ['admin']
E       User roles: ['admin']
E       Required permission: Permission.DELETE
E       Resource owner: alice, Current user: bob
E       assert False

test_permissions.py:18: AssertionError
========================= 1 failed in 0.09s ==========================
```

**Expected vs. Actual improvement**: The custom message immediately tells us:
- What should have happened: "Admin should be able to delete any resource"
- What actually happened: "got: insufficient_permissions"
- All relevant context: user roles, required permission, resource owner, current user

Without the custom message, we'd only see `assert False`, which provides no context.

## Best Practices for Custom Messages

### 1. Include Relevant Context

Always include the values that led to the failure:

```python
# Good: Includes context
assert result > threshold, (
    f"Result {result} should exceed threshold {threshold}\n"
    f"Input values: x={x}, y={y}"
)

# Bad: No context
assert result > threshold, "Result too low"
```

### 2. Explain the Expectation

State what should have happened, not just what went wrong:

```python
# Good: Explains expectation
assert user.is_active, (
    f"User {user.username} should be active after email verification, "
    f"but is_active={user.is_active}"
)

# Bad: Just states the failure
assert user.is_active, "User not active"
```

### 3. Use Multi-line Strings for Readability

For complex messages, use parentheses and line breaks:

```python
# Good: Readable multi-line message
assert actual == expected, (
    f"Configuration mismatch:\n"
    f"Expected: {expected}\n"
    f"Actual: {actual}\n"
    f"Difference: {set(expected.keys()) ^ set(actual.keys())}"
)

# Bad: Hard to read single line
assert actual == expected, f"Config mismatch: expected {expected} but got {actual} with diff {set(expected.keys()) ^ set(actual.keys())}"
```

### 4. Don't Duplicate Introspection

Pytest already shows the values being compared, so don't just repeat them:

```python
# Bad: Duplicates what pytest shows
assert x == y, f"x ({x}) should equal y ({y})"

# Good: Adds context pytest doesn't have
assert x == y, f"Calculation result should match expected value for input={input_val}"
```

### 5. Use Custom Messages for Complex Logic

When the assertion involves complex logic, explain the business rule:

```python
# Good: Explains the business rule
assert (
    user.age >= 18 or user.has_parental_consent
), (
    f"User {user.username} (age {user.age}) must be 18+ or have parental consent. "
    f"has_parental_consent={user.has_parental_consent}"
)

# Bad: Just restates the code
assert user.age >= 18 or user.has_parental_consent, "Age or consent check failed"
```

## When to Use Custom Messages

**Use custom messages when**:
- The assertion logic is complex or non-obvious
- You need to explain business rules or domain concepts
- Multiple conditions are being checked
- The failure could have multiple causes
- You want to guide the developer toward the fix

**Don't use custom messages when**:
- Simple equality checks where introspection is sufficient
- The assertion is self-explanatory
- You're just repeating what pytest already shows

## Approximate Comparisons with pytest.approx

Earlier we saw floating point comparison fail. Here's the solution:

```python
# test_floating_point_fixed.py
import pytest

def test_calculation_result_with_approx():
    """Test floating point calculation with approximate comparison"""
    result = 0.1 + 0.2
    expected = 0.3
    
    assert result == pytest.approx(expected)

def test_calculation_with_custom_tolerance():
    """Test with custom tolerance"""
    result = 0.1 + 0.2
    expected = 0.3
    
    # Allow 1% relative difference or 0.01 absolute difference
    assert result == pytest.approx(expected, rel=1e-2, abs=1e-2)

def test_list_of_floats():
    """Test that pytest.approx works with collections"""
    results = [0.1 + 0.2, 0.2 + 0.3, 0.3 + 0.4]
    expected = [0.3, 0.5, 0.7]
    
    assert results == pytest.approx(expected)
```

Run the tests:

```bash
pytest test_floating_point_fixed.py -v
```

**Output**:
```
======================== test session starts =========================
collected 3 items

test_floating_point_fixed.py::test_calculation_result_with_approx PASSED [33%]
test_floating_point_fixed.py::test_calculation_with_custom_tolerance PASSED [66%]
test_floating_point_fixed.py::test_list_of_floats PASSED        [100%]

========================= 3 passed in 0.06s ==========================
```

`pytest.approx()` handles floating point comparison with configurable tolerance, and works with numbers, lists, tuples, and dictionaries containing numbers.

## Testing Exceptions with pytest.raises()

## Deep Dive: Exception Testing Patterns

We introduced `pytest.raises()` in Section 7.1, but there's much more to learn about testing exceptions effectively. Let's explore advanced patterns and common pitfalls.

## Iteration 7: Verifying Exception Attributes

Beyond checking exception type and message, you often need to verify custom exception attributes:

```python
# payment_processing.py
from decimal import Decimal
from typing import Optional

class PaymentError(Exception):
    """Base exception for payment failures"""
    def __init__(
        self,
        message: str,
        error_code: str,
        amount: Optional[Decimal] = None,
        transaction_id: Optional[str] = None
    ):
        super().__init__(message)
        self.error_code = error_code
        self.amount = amount
        self.transaction_id = transaction_id

class InsufficientFundsError(PaymentError):
    """Raised when account has insufficient funds"""
    def __init__(
        self,
        required: Decimal,
        available: Decimal,
        transaction_id: str
    ):
        message = f"Insufficient funds: need {required}, have {available}"
        super().__init__(
            message=message,
            error_code="INSUFFICIENT_FUNDS",
            amount=required,
            transaction_id=transaction_id
        )
        self.required = required
        self.available = available

class InvalidCardError(PaymentError):
    """Raised when card is invalid"""
    def __init__(self, card_number: str, reason: str):
        message = f"Invalid card {card_number}: {reason}"
        super().__init__(
            message=message,
            error_code="INVALID_CARD"
        )
        self.card_number = card_number
        self.reason = reason

def process_payment(
    amount: Decimal,
    card_number: str,
    account_balance: Decimal
) -> str:
    """
    Process a payment.
    Returns transaction_id on success.
    Raises PaymentError on failure.
    """
    # Validate card
    if not card_number.isdigit() or len(card_number) != 16:
        raise InvalidCardError(
            card_number=card_number,
            reason="must be 16 digits"
        )
    
    # Check funds
    if account_balance < amount:
        transaction_id = f"TXN_{card_number[:4]}_{amount}"
        raise InsufficientFundsError(
            required=amount,
            available=account_balance,
            transaction_id=transaction_id
        )
    
    # Process payment
    transaction_id = f"TXN_{card_number[:4]}_{amount}_SUCCESS"
    return transaction_id
```

Now let's write tests that verify exception attributes:

```python
# test_payment_v1.py
import pytest
from decimal import Decimal
from payment_processing import (
    process_payment,
    InsufficientFundsError,
    InvalidCardError
)

def test_insufficient_funds_exception_attributes():
    """Test that InsufficientFundsError contains correct attributes"""
    with pytest.raises(InsufficientFundsError) as exc_info:
        process_payment(
            amount=Decimal("100.00"),
            card_number="1234567890123456",
            account_balance=Decimal("50.00")
        )
    
    # Access the exception object
    exc = exc_info.value
    
    # Verify all attributes
    assert exc.error_code == "INSUFFICIENT_FUNDS"
    assert exc.required == Decimal("100.00")
    assert exc.available == Decimal("50.00")
    assert exc.amount == Decimal("100.00")
    assert exc.transaction_id.startswith("TXN_1234")
    
    # Verify message format
    assert "need 100.00" in str(exc)
    assert "have 50.00" in str(exc)

def test_invalid_card_exception_attributes():
    """Test that InvalidCardError contains correct attributes"""
    with pytest.raises(InvalidCardError) as exc_info:
        process_payment(
            amount=Decimal("100.00"),
            card_number="123",  # Too short
            account_balance=Decimal("200.00")
        )
    
    exc = exc_info.value
    
    assert exc.error_code == "INVALID_CARD"
    assert exc.card_number == "123"
    assert exc.reason == "must be 16 digits"
    assert "Invalid card 123" in str(exc)
```

Run the tests:

```bash
pytest test_payment_v1.py -v
```

**Output**:
```
======================== test session starts =========================
collected 2 items

test_payment_v1.py::test_insufficient_funds_exception_attributes PASSED [50%]
test_payment_v1.py::test_invalid_card_exception_attributes PASSED [100%]

========================= 2 passed in 0.07s ==========================
```

**Key pattern**: Use `exc_info.value` to access the exception object, then assert on its attributes just like any other object.

## Iteration 8: Testing Exception Inheritance

Sometimes you need to verify that the right exception type is raised, considering inheritance:

```python
# test_payment_v2.py
import pytest
from decimal import Decimal
from payment_processing import (
    process_payment,
    PaymentError,
    InsufficientFundsError,
    InvalidCardError
)

def test_insufficient_funds_is_payment_error():
    """Test that InsufficientFundsError is a subclass of PaymentError"""
    with pytest.raises(PaymentError) as exc_info:
        process_payment(
            amount=Decimal("100.00"),
            card_number="1234567890123456",
            account_balance=Decimal("50.00")
        )
    
    # Verify it's specifically InsufficientFundsError
    assert isinstance(exc_info.value, InsufficientFundsError)
    assert exc_info.value.error_code == "INSUFFICIENT_FUNDS"

def test_invalid_card_is_payment_error():
    """Test that InvalidCardError is a subclass of PaymentError"""
    with pytest.raises(PaymentError) as exc_info:
        process_payment(
            amount=Decimal("100.00"),
            card_number="invalid",
            account_balance=Decimal("200.00")
        )
    
    # Verify it's specifically InvalidCardError
    assert isinstance(exc_info.value, InvalidCardError)
    assert exc_info.value.error_code == "INVALID_CARD"
```

Run the tests:

```bash
pytest test_payment_v2.py -v
```

**Output**:
```
======================== test session starts =========================
collected 2 items

test_payment_v2.py::test_insufficient_funds_is_payment_error PASSED [50%]
test_payment_v2.py::test_invalid_card_is_payment_error PASSED [100%]

========================= 2 passed in 0.06s ==========================
```

**Key insight**: `pytest.raises(BaseException)` will catch any subclass of `BaseException`. Use `isinstance()` to verify the specific subclass when needed.

## Iteration 9: Testing Exception Messages with Regex

Sometimes you need more flexible message matching than substring search:

```python
# test_payment_v3.py
import pytest
import re
from decimal import Decimal
from payment_processing import (
    process_payment,
    InsufficientFundsError
)

def test_insufficient_funds_message_format():
    """Test that error message follows expected format"""
    with pytest.raises(
        InsufficientFundsError,
        match=r"Insufficient funds: need \d+\.\d+, have \d+\.\d+"
    ) as exc_info:
        process_payment(
            amount=Decimal("100.00"),
            card_number="1234567890123456",
            account_balance=Decimal("50.00")
        )

def test_invalid_card_message_format():
    """Test that invalid card message follows expected format"""
    with pytest.raises(
        InvalidCardError,
        match=r"Invalid card \d+: must be 16 digits"
    ) as exc_info:
        process_payment(
            amount=Decimal("100.00"),
            card_number="123",
            account_balance=Decimal("200.00")
        )
```

Run the tests:

```bash
pytest test_payment_v3.py -v
```

**Output**:
```
======================== test session starts =========================
collected 2 items

test_payment_v3.py::test_insufficient_funds_message_format PASSED [50%]
test_payment_v3.py::test_invalid_card_message_format PASSED [100%]

========================= 2 passed in 0.07s ==========================
```

The `match` parameter accepts a regex pattern. The test passes if the exception message matches the pattern.

Let's see what happens when the pattern doesn't match:

```python
# test_payment_v3_failing.py
import pytest
from decimal import Decimal
from payment_processing import process_payment, InsufficientFundsError

def test_insufficient_funds_wrong_pattern():
    """Test with wrong regex pattern"""
    with pytest.raises(
        InsufficientFundsError,
        match=r"Account balance too low"  # Wrong pattern
    ):
        process_payment(
            amount=Decimal("100.00"),
            card_number="1234567890123456",
            account_balance=Decimal("50.00")
        )
```

Run the failing test:

```bash
pytest test_payment_v3_failing.py -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_payment_v3_failing.py::test_insufficient_funds_wrong_pattern FAILED [100%]

============================== FAILURES ==============================
__________ test_insufficient_funds_wrong_pattern _____________________

    def test_insufficient_funds_wrong_pattern():
        with pytest.raises(
            InsufficientFundsError,
            match=r"Account balance too low"
        ):
>           process_payment(
                amount=Decimal("100.00"),
                card_number="1234567890123456",
                account_balance=Decimal("50.00")
            )
E           AssertionError: Pattern 'Account balance too low' does not match 'Insufficient funds: need 100.00, have 50.00'

test_payment_v3_failing.py:10: AssertionError
========================= 1 failed in 0.08s ==========================
```

### Diagnostic Analysis: Match Parameter Failure

**The output shows**:
- The pattern we expected: `'Account balance too low'`
- The actual message: `'Insufficient funds: need 100.00, have 50.00'`
- Clear indication that the pattern didn't match

**Key insight**: The `match` parameter is useful for verifying message format without hardcoding exact values.

## Common Failure Modes and Their Signatures

### Symptom: Test passes when no exception is raised

**Pytest output pattern**:
```
FAILED test_file.py::test_name - Failed: DID NOT RAISE <class 'ExceptionType'>
```

**Diagnostic clues**:
- Message explicitly states "DID NOT RAISE"
- Shows the expected exception type

**Root cause**: The code path didn't raise the expected exception

**Solution**: Verify your test inputs actually trigger the error condition

```python
# Example of this failure
def test_invalid_email_should_raise():
    """This test will fail - email is actually valid"""
    with pytest.raises(InvalidEmailError):
        register_user(
            email="valid@example.com",  # Bug: this is valid!
            username="test",
            password="SecurePass123"
        )
```

### Symptom: Wrong exception type is raised

**Pytest output pattern**:
```
FAILED test_file.py::test_name - ValueError: some message
```

**Diagnostic clues**:
- Shows the actual exception type that was raised
- No mention of "DID NOT RAISE"

**Root cause**: Code raises a different exception than expected

**Solution**: Either fix the code or update the test to expect the correct exception type

```python
# Example of this failure
def test_expects_custom_but_gets_builtin():
    """This test will fail - raises ValueError, not CustomError"""
    with pytest.raises(CustomError):
        # Code actually raises ValueError
        int("not a number")
```

### Symptom: Exception message doesn't match pattern

**Pytest output pattern**:
```
AssertionError: Pattern 'expected pattern' does not match 'actual message'
```

**Diagnostic clues**:
- Shows both the expected pattern and actual message
- Indicates regex match failure

**Root cause**: Exception message format changed or pattern is incorrect

**Solution**: Update the pattern or fix the message format

## Testing Multiple Exceptions in One Test

Sometimes you need to test that different inputs raise different exceptions:

```python
# test_payment_v4.py
import pytest
from decimal import Decimal
from payment_processing import (
    process_payment,
    InsufficientFundsError,
    InvalidCardError
)

def test_various_payment_failures():
    """Test that different invalid inputs raise appropriate exceptions"""
    
    # Test insufficient funds
    with pytest.raises(InsufficientFundsError):
        process_payment(
            amount=Decimal("100.00"),
            card_number="1234567890123456",
            account_balance=Decimal("50.00")
        )
    
    # Test invalid card - too short
    with pytest.raises(InvalidCardError) as exc_info:
        process_payment(
            amount=Decimal("100.00"),
            card_number="123",
            account_balance=Decimal("200.00")
        )
    assert "must be 16 digits" in str(exc_info.value)
    
    # Test invalid card - non-numeric
    with pytest.raises(InvalidCardError) as exc_info:
        process_payment(
            amount=Decimal("100.00"),
            card_number="abcd567890123456",
            account_balance=Decimal("200.00")
        )
    assert "must be 16 digits" in str(exc_info.value)
```

**Pattern**: Multiple `with pytest.raises()` blocks in one test function. Each block tests a different error condition.

**When to use this**: When testing related error conditions that share setup logic.

**When to avoid this**: When error conditions are unrelated—separate tests are clearer.

## Testing That Exceptions Are NOT Raised

Sometimes you need to verify that code completes successfully without raising:

```python
# test_payment_v5.py
import pytest
from decimal import Decimal
from payment_processing import process_payment

def test_successful_payment_no_exception():
    """Test that valid payment completes without exception"""
    # This is the pattern: just call the function
    # If it raises, the test fails
    transaction_id = process_payment(
        amount=Decimal("100.00"),
        card_number="1234567890123456",
        account_balance=Decimal("200.00")
    )
    
    # Then verify the result
    assert transaction_id.startswith("TXN_1234")
    assert "SUCCESS" in transaction_id
```

**Key insight**: You don't need special syntax to test that exceptions aren't raised. Just call the function normally—if it raises, the test fails automatically.

## Advanced Pattern: Parametrizing Exception Tests

When you have multiple similar exception scenarios, use parametrization:

```python
# test_payment_v6.py
import pytest
from decimal import Decimal
from payment_processing import (
    process_payment,
    InvalidCardError
)

@pytest.mark.parametrize("card_number,expected_reason", [
    ("123", "must be 16 digits"),
    ("12345678901234567", "must be 16 digits"),  # Too long
    ("abcd567890123456", "must be 16 digits"),  # Non-numeric
    ("", "must be 16 digits"),  # Empty
])
def test_invalid_card_numbers(card_number, expected_reason):
    """Test that various invalid card numbers raise InvalidCardError"""
    with pytest.raises(InvalidCardError) as exc_info:
        process_payment(
            amount=Decimal("100.00"),
            card_number=card_number,
            account_balance=Decimal("200.00")
        )
    
    assert exc_info.value.card_number == card_number
    assert expected_reason in str(exc_info.value)
```

Run the parametrized tests:

```bash
pytest test_payment_v6.py -v
```

**Output**:
```
======================== test session starts =========================
collected 4 items

test_payment_v6.py::test_invalid_card_numbers[123-must be 16 digits] PASSED [25%]
test_payment_v6.py::test_invalid_card_numbers[12345678901234567-must be 16 digits] PASSED [50%]
test_payment_v6.py::test_invalid_card_numbers[abcd567890123456-must be 16 digits] PASSED [75%]
test_payment_v6.py::test_invalid_card_numbers[-must be 16 digits] PASSED [100%]

========================= 4 passed in 0.09s ==========================
```

This pattern efficiently tests multiple error conditions with minimal code duplication.

## Testing Warnings with pytest.warns()

## Understanding Python Warnings

Python's warning system is designed for non-fatal issues that developers should know about but that don't prevent code execution. Common use cases:

- Deprecation notices
- Performance concerns
- Potential misuse of APIs
- Configuration issues

Testing warnings ensures your code properly alerts users to these issues.

## Iteration 10: Basic Warning Testing

Let's create a system that issues warnings for deprecated features:

```python
# api_client.py
import warnings
from typing import Optional, Dict, Any

class DeprecationWarning(UserWarning):
    """Warning for deprecated features"""
    pass

class APIClient:
    def __init__(self, api_key: str, base_url: str = "https://api.example.com"):
        self.api_key = api_key
        self.base_url = base_url
    
    def fetch_user(self, user_id: int) -> Dict[str, Any]:
        """
        Fetch user data.
        
        DEPRECATED: Use fetch_user_v2() instead.
        This method will be removed in version 3.0.
        """
        warnings.warn(
            "fetch_user() is deprecated, use fetch_user_v2() instead",
            DeprecationWarning,
            stacklevel=2
        )
        return {"id": user_id, "name": "User"}
    
    def fetch_user_v2(self, user_id: int) -> Dict[str, Any]:
        """Fetch user data (new version)"""
        return {"id": user_id, "name": "User", "version": 2}
    
    def get_data(
        self,
        endpoint: str,
        use_cache: bool = True,
        cache_ttl: Optional[int] = None
    ) -> Dict[str, Any]:
        """
        Fetch data from endpoint.
        
        Args:
            endpoint: API endpoint
            use_cache: Whether to use cache
            cache_ttl: Cache TTL in seconds (deprecated parameter)
        """
        if cache_ttl is not None:
            warnings.warn(
                "cache_ttl parameter is deprecated and will be ignored. "
                "Use cache configuration instead.",
                DeprecationWarning,
                stacklevel=2
            )
        
        return {"endpoint": endpoint, "cached": use_cache}
```

Now let's write tests for these warnings:

```python
# test_warnings_v1.py
import pytest
import warnings
from api_client import APIClient, DeprecationWarning

def test_fetch_user_issues_deprecation_warning():
    """Test that fetch_user() issues a deprecation warning"""
    client = APIClient(api_key="test-key")
    
    with pytest.warns(DeprecationWarning):
        client.fetch_user(user_id=123)

def test_fetch_user_v2_no_warning():
    """Test that fetch_user_v2() does not issue a warning"""
    client = APIClient(api_key="test-key")
    
    # This should complete without warnings
    result = client.fetch_user_v2(user_id=123)
    assert result["version"] == 2

def test_cache_ttl_parameter_warning():
    """Test that cache_ttl parameter issues a warning"""
    client = APIClient(api_key="test-key")
    
    with pytest.warns(DeprecationWarning):
        client.get_data(endpoint="/users", cache_ttl=300)
```

Run the tests:

```bash
pytest test_warnings_v1.py -v
```

**Output**:
```
======================== test session starts =========================
collected 3 items

test_warnings_v1.py::test_fetch_user_issues_deprecation_warning PASSED [33%]
test_warnings_v1.py::test_fetch_user_v2_no_warning PASSED       [66%]
test_warnings_v1.py::test_cache_ttl_parameter_warning PASSED    [100%]

========================= 3 passed in 0.08s ==========================
```

**Key pattern**: `pytest.warns()` works just like `pytest.raises()`—it's a context manager that captures warnings.

## Iteration 11: Inspecting Warning Messages

Just like with exceptions, you often need to verify the warning message:

```python
# test_warnings_v2.py
import pytest
from api_client import APIClient, DeprecationWarning

def test_fetch_user_warning_message():
    """Test that deprecation warning has correct message"""
    client = APIClient(api_key="test-key")
    
    with pytest.warns(DeprecationWarning) as warning_info:
        client.fetch_user(user_id=123)
    
    # Access the warning message
    assert len(warning_info) == 1
    warning = warning_info[0]
    assert "fetch_user() is deprecated" in str(warning.message)
    assert "fetch_user_v2()" in str(warning.message)

def test_cache_ttl_warning_message():
    """Test that cache_ttl warning has correct message"""
    client = APIClient(api_key="test-key")
    
    with pytest.warns(DeprecationWarning) as warning_info:
        client.get_data(endpoint="/users", cache_ttl=300)
    
    assert len(warning_info) == 1
    warning = warning_info[0]
    assert "cache_ttl parameter is deprecated" in str(warning.message)
    assert "cache configuration" in str(warning.message)
```

Run the tests:

```bash
pytest test_warnings_v2.py -v
```

**Output**:
```
======================== test session starts =========================
collected 2 items

test_warnings_v2.py::test_fetch_user_warning_message PASSED     [50%]
test_warnings_v2.py::test_cache_ttl_warning_message PASSED      [100%]

========================= 2 passed in 0.07s ==========================
```

### Understanding WarningInfo

When you use `with pytest.warns() as warning_info:`, the `warning_info` object is a list of warnings. Each warning has:

- `message`: The warning message
- `category`: The warning class
- `filename`: Where the warning was issued
- `lineno`: Line number where the warning was issued

## Iteration 12: Testing Multiple Warnings

Sometimes code issues multiple warnings:

```python
# api_client_extended.py
import warnings
from typing import Optional, Dict, Any

class APIClientExtended:
    def __init__(self, api_key: str):
        self.api_key = api_key
    
    def complex_operation(
        self,
        param1: str,
        param2: Optional[int] = None,
        param3: Optional[str] = None,
        param4: Optional[bool] = None
    ) -> Dict[str, Any]:
        """
        Complex operation with multiple deprecated parameters.
        """
        warnings_issued = []
        
        if param2 is not None:
            warnings.warn(
                "param2 is deprecated",
                DeprecationWarning,
                stacklevel=2
            )
            warnings_issued.append("param2")
        
        if param3 is not None:
            warnings.warn(
                "param3 is deprecated",
                DeprecationWarning,
                stacklevel=2
            )
            warnings_issued.append("param3")
        
        if param4 is not None:
            warnings.warn(
                "param4 is deprecated",
                DeprecationWarning,
                stacklevel=2
            )
            warnings_issued.append("param4")
        
        return {
            "param1": param1,
            "warnings": warnings_issued
        }
```

Test multiple warnings:

```python
# test_warnings_v3.py
import pytest
from api_client_extended import APIClientExtended, DeprecationWarning

def test_multiple_deprecated_parameters():
    """Test that using multiple deprecated parameters issues multiple warnings"""
    client = APIClientExtended(api_key="test-key")
    
    with pytest.warns(DeprecationWarning) as warning_info:
        client.complex_operation(
            param1="value1",
            param2=42,
            param3="value3",
            param4=True
        )
    
    # Verify we got exactly 3 warnings
    assert len(warning_info) == 3
    
    # Verify each warning message
    messages = [str(w.message) for w in warning_info]
    assert "param2 is deprecated" in messages[0]
    assert "param3 is deprecated" in messages[1]
    assert "param4 is deprecated" in messages[2]

def test_single_deprecated_parameter():
    """Test that using one deprecated parameter issues one warning"""
    client = APIClientExtended(api_key="test-key")
    
    with pytest.warns(DeprecationWarning) as warning_info:
        client.complex_operation(
            param1="value1",
            param2=42
        )
    
    assert len(warning_info) == 1
    assert "param2 is deprecated" in str(warning_info[0].message)
```

Run the tests:

```bash
pytest test_warnings_v3.py -v
```

**Output**:
```
======================== test session starts =========================
collected 2 items

test_warnings_v3.py::test_multiple_deprecated_parameters PASSED [50%]
test_warnings_v3.py::test_single_deprecated_parameter PASSED    [100%]

========================= 2 passed in 0.08s ==========================
```

## Testing That Warnings Are NOT Issued

To verify that code doesn't issue warnings, use `warnings.simplefilter()`:

```python
# test_warnings_v4.py
import pytest
import warnings
from api_client import APIClient

def test_fetch_user_v2_no_warnings():
    """Test that fetch_user_v2() issues no warnings"""
    client = APIClient(api_key="test-key")
    
    # Turn warnings into errors for this test
    with warnings.catch_warnings():
        warnings.simplefilter("error")
        
        # If any warning is issued, this will raise an exception
        result = client.fetch_user_v2(user_id=123)
        assert result["version"] == 2

def test_get_data_without_deprecated_params():
    """Test that get_data() without deprecated params issues no warnings"""
    client = APIClient(api_key="test-key")
    
    with warnings.catch_warnings():
        warnings.simplefilter("error")
        
        result = client.get_data(endpoint="/users", use_cache=True)
        assert result["endpoint"] == "/users"
```

Run the tests:

```bash
pytest test_warnings_v4.py -v
```

**Output**:
```
======================== test session starts =========================
collected 2 items

test_warnings_v4.py::test_fetch_user_v2_no_warnings PASSED      [50%]
test_warnings_v4.py::test_get_data_without_deprecated_params PASSED [100%]

========================= 2 passed in 0.07s ==========================
```

**Key pattern**: `warnings.simplefilter("error")` converts warnings to exceptions, so any warning will fail the test.

## Using match Parameter with pytest.warns()

Just like `pytest.raises()`, `pytest.warns()` accepts a `match` parameter for regex matching:

```python
# test_warnings_v5.py
import pytest
from api_client import APIClient, DeprecationWarning

def test_warning_message_format():
    """Test that warning message follows expected format"""
    client = APIClient(api_key="test-key")
    
    with pytest.warns(
        DeprecationWarning,
        match=r"fetch_user\(\) is deprecated, use fetch_user_v2\(\) instead"
    ):
        client.fetch_user(user_id=123)

def test_cache_ttl_warning_format():
    """Test cache_ttl warning message format"""
    client = APIClient(api_key="test-key")
    
    with pytest.warns(
        DeprecationWarning,
        match=r"cache_ttl parameter is deprecated.*cache configuration"
    ):
        client.get_data(endpoint="/users", cache_ttl=300)
```

Run the tests:

```bash
pytest test_warnings_v5.py -v
```

**Output**:
```
======================== test session starts =========================
collected 2 items

test_warnings_v5.py::test_warning_message_format PASSED         [50%]
test_warnings_v5.py::test_cache_ttl_warning_format PASSED       [100%]

========================= 2 passed in 0.07s ==========================
```

## Common Warning Testing Pitfalls

### Pitfall 1: Warnings Not Captured

If your test doesn't capture a warning you expect:

```python
# test_warnings_pitfall1.py
import pytest
import warnings
from api_client import APIClient, DeprecationWarning

def test_warning_not_captured():
    """This test will fail - warning is issued before the context manager"""
    client = APIClient(api_key="test-key")
    
    # BUG: Warning issued here, outside the context manager
    client.fetch_user(user_id=123)
    
    # Context manager expects a warning but won't see it
    with pytest.warns(DeprecationWarning):
        pass  # No warning issued here!
```

Run the failing test:

```bash
pytest test_warnings_pitfall1.py -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_warnings_pitfall1.py::test_warning_not_captured FAILED    [100%]

============================== FAILURES ==============================
_________________ test_warning_not_captured __________________________

    def test_warning_not_captured():
        client = APIClient(api_key="test-key")
        client.fetch_user(user_id=123)
        
>       with pytest.warns(DeprecationWarning):
            pass
E       Failed: DID NOT WARN. No warnings of type (<class 'api_client.DeprecationWarning'>,) were emitted.

test_warnings_pitfall1.py:12: Failed
========================= 1 failed in 0.09s ==========================
```

**Solution**: Ensure the code that issues the warning is inside the `with pytest.warns()` block.

### Pitfall 2: Wrong Warning Category

If you expect the wrong warning type:

```python
# test_warnings_pitfall2.py
import pytest
from api_client import APIClient

def test_wrong_warning_type():
    """This test will fail - expects UserWarning but gets DeprecationWarning"""
    client = APIClient(api_key="test-key")
    
    with pytest.warns(UserWarning):  # Wrong: should be DeprecationWarning
        client.fetch_user(user_id=123)
```

Run the failing test:

```bash
pytest test_warnings_pitfall2.py -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_warnings_pitfall2.py::test_wrong_warning_type FAILED       [100%]

============================== FAILURES ==============================
__________________ test_wrong_warning_type ___________________________

    def test_wrong_warning_type():
        client = APIClient(api_key="test-key")
        
>       with pytest.warns(UserWarning):
            client.fetch_user(user_id=123)
E       Failed: DID NOT WARN. No warnings of type (<class 'UserWarning'>,) were emitted. The list of emitted warnings is: [DeprecationWarning('fetch_user() is deprecated, use fetch_user_v2() instead')].

test_warnings_pitfall2.py:8: Failed
========================= 1 failed in 0.09s ==========================
```

**Diagnostic clues**: The error message helpfully shows what warnings *were* emitted, making it easy to fix the test.

## When to Test Warnings

**Test warnings when**:
- You're deprecating features and want to ensure users are notified
- You have configuration issues that should warn but not fail
- You're alerting users to performance concerns
- You want to ensure warnings are issued at the right time

**Don't test warnings when**:
- The warning comes from a third-party library you don't control
- The warning is purely informational with no action required
- You're testing that warnings don't occur (use `warnings.simplefilter("error")` instead)

## Context Managers for Advanced Assertions

## Beyond pytest.raises() and pytest.warns()

We've seen how `pytest.raises()` and `pytest.warns()` use context managers to capture and verify exceptions and warnings. Now let's explore how to create custom context managers for more sophisticated assertion patterns.

## Iteration 13: Testing State Changes

Let's create a system that tracks state changes and verify they happen correctly:

```python
# state_machine.py
from enum import Enum
from typing import List, Optional
from datetime import datetime

class OrderStatus(Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

class InvalidTransitionError(Exception):
    """Raised when state transition is invalid"""
    pass

class Order:
    def __init__(self, order_id: str):
        self.order_id = order_id
        self.status = OrderStatus.PENDING
        self.history: List[tuple[OrderStatus, datetime]] = [
            (OrderStatus.PENDING, datetime.now())
        ]
    
    def transition_to(self, new_status: OrderStatus) -> None:
        """
        Transition to new status.
        Raises InvalidTransitionError if transition is not allowed.
        """
        valid_transitions = {
            OrderStatus.PENDING: {OrderStatus.PROCESSING, OrderStatus.CANCELLED},
            OrderStatus.PROCESSING: {OrderStatus.SHIPPED, OrderStatus.CANCELLED},
            OrderStatus.SHIPPED: {OrderStatus.DELIVERED},
            OrderStatus.DELIVERED: set(),
            OrderStatus.CANCELLED: set(),
        }
        
        if new_status not in valid_transitions[self.status]:
            raise InvalidTransitionError(
                f"Cannot transition from {self.status.value} to {new_status.value}"
            )
        
        self.status = new_status
        self.history.append((new_status, datetime.now()))
    
    def get_status_history(self) -> List[OrderStatus]:
        """Return list of statuses in order"""
        return [status for status, _ in self.history]
```

Let's write tests that verify state changes:

```python
# test_state_v1.py
import pytest
from state_machine import Order, OrderStatus, InvalidTransitionError

def test_valid_order_lifecycle():
    """Test complete order lifecycle"""
    order = Order(order_id="ORD-001")
    
    # Initial state
    assert order.status == OrderStatus.PENDING
    
    # Transition to processing
    order.transition_to(OrderStatus.PROCESSING)
    assert order.status == OrderStatus.PROCESSING
    
    # Transition to shipped
    order.transition_to(OrderStatus.SHIPPED)
    assert order.status == OrderStatus.SHIPPED
    
    # Transition to delivered
    order.transition_to(OrderStatus.DELIVERED)
    assert order.status == OrderStatus.DELIVERED
    
    # Verify history
    expected_history = [
        OrderStatus.PENDING,
        OrderStatus.PROCESSING,
        OrderStatus.SHIPPED,
        OrderStatus.DELIVERED
    ]
    assert order.get_status_history() == expected_history

def test_invalid_transition():
    """Test that invalid transition raises error"""
    order = Order(order_id="ORD-002")
    
    with pytest.raises(InvalidTransitionError) as exc_info:
        # Cannot go directly from PENDING to SHIPPED
        order.transition_to(OrderStatus.SHIPPED)
    
    assert "Cannot transition from pending to shipped" in str(exc_info.value)
    # Verify state didn't change
    assert order.status == OrderStatus.PENDING
```

Run the tests:

```bash
pytest test_state_v1.py -v
```

**Output**:
```
======================== test session starts =========================
collected 2 items

test_state_v1.py::test_valid_order_lifecycle PASSED             [50%]
test_state_v1.py::test_invalid_transition PASSED                [100%]

========================= 2 passed in 0.08s ==========================
```

**Current limitation**: The test is verbose and repetitive. Each state transition requires:
1. Call `transition_to()`
2. Assert the new status
3. Optionally verify the transition succeeded

What if we could create a context manager that automatically verifies state changes?

## Creating a Custom Context Manager for State Verification

Let's create a context manager that captures the before and after state:

```python
# test_helpers.py
from contextlib import contextmanager
from typing import Any, Callable

@contextmanager
def assert_state_changes(
    obj: Any,
    attribute: str,
    expected_before: Any,
    expected_after: Any
):
    """
    Context manager that verifies an attribute changes from one value to another.
    
    Usage:
        with assert_state_changes(order, 'status', OrderStatus.PENDING, OrderStatus.PROCESSING):
            order.transition_to(OrderStatus.PROCESSING)
    """
    # Verify initial state
    actual_before = getattr(obj, attribute)
    assert actual_before == expected_before, (
        f"Expected {attribute} to be {expected_before} before operation, "
        f"but was {actual_before}"
    )
    
    # Yield control to the test
    yield
    
    # Verify final state
    actual_after = getattr(obj, attribute)
    assert actual_after == expected_after, (
        f"Expected {attribute} to be {expected_after} after operation, "
        f"but was {actual_after}"
    )
```

Now let's use this context manager in our tests:

```python
# test_state_v2.py
import pytest
from state_machine import Order, OrderStatus
from test_helpers import assert_state_changes

def test_order_lifecycle_with_context_manager():
    """Test order lifecycle using custom context manager"""
    order = Order(order_id="ORD-003")
    
    # Each transition is verified automatically
    with assert_state_changes(order, 'status', OrderStatus.PENDING, OrderStatus.PROCESSING):
        order.transition_to(OrderStatus.PROCESSING)
    
    with assert_state_changes(order, 'status', OrderStatus.PROCESSING, OrderStatus.SHIPPED):
        order.transition_to(OrderStatus.SHIPPED)
    
    with assert_state_changes(order, 'status', OrderStatus.SHIPPED, OrderStatus.DELIVERED):
        order.transition_to(OrderStatus.DELIVERED)

def test_cancelled_order():
    """Test cancelling an order"""
    order = Order(order_id="ORD-004")
    
    with assert_state_changes(order, 'status', OrderStatus.PENDING, OrderStatus.CANCELLED):
        order.transition_to(OrderStatus.CANCELLED)
```

Run the tests:

```bash
pytest test_state_v2.py -v
```

**Output**:
```
======================== test session starts =========================
collected 2 items

test_state_v2.py::test_order_lifecycle_with_context_manager PASSED [50%]
test_state_v2.py::test_cancelled_order PASSED                   [100%]

========================= 2 passed in 0.07s ==========================
```

**Expected vs. Actual improvement**: The tests are more concise and the intent is clearer. The context manager handles both the before and after verification.

Let's see what happens when a state change fails:

```python
# test_state_v2_failing.py
import pytest
from state_machine import Order, OrderStatus
from test_helpers import assert_state_changes

def test_state_change_fails():
    """Test what happens when state doesn't change as expected"""
    order = Order(order_id="ORD-005")
    
    # This will fail - we expect SHIPPED but will get PROCESSING
    with assert_state_changes(order, 'status', OrderStatus.PENDING, OrderStatus.SHIPPED):
        order.transition_to(OrderStatus.PROCESSING)  # Wrong transition!
```

Run the failing test:

```bash
pytest test_state_v2_failing.py -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_state_v2_failing.py::test_state_change_fails FAILED        [100%]

============================== FAILURES ==============================
__________________ test_state_change_fails ___________________________

    def test_state_change_fails():
        order = Order(order_id="ORD-005")
        
        with assert_state_changes(order, 'status', OrderStatus.PENDING, OrderStatus.SHIPPED):
>           order.transition_to(OrderStatus.PROCESSING)

test_helpers.py:28: AssertionError
E       AssertionError: Expected status to be OrderStatus.SHIPPED after operation, but was OrderStatus.PROCESSING

test_state_v2_failing.py:9: 
========================= 1 failed in 0.09s ==========================
```

**Diagnostic Analysis**: The context manager's custom message clearly explains:
- What attribute was being checked: `status`
- What value was expected: `OrderStatus.SHIPPED`
- What value was actually found: `OrderStatus.PROCESSING`

## Iteration 14: Testing Resource Cleanup

Context managers are perfect for testing that resources are properly cleaned up:

```python
# resource_manager.py
from typing import Optional
import tempfile
import os

class FileResource:
    """Manages a temporary file resource"""
    def __init__(self, filename: str):
        self.filename = filename
        self.file_handle: Optional[object] = None
        self.is_open = False
    
    def open(self) -> None:
        """Open the file resource"""
        self.file_handle = open(self.filename, 'w')
        self.is_open = True
    
    def write(self, content: str) -> None:
        """Write to the file"""
        if not self.is_open:
            raise RuntimeError("Resource not open")
        self.file_handle.write(content)
    
    def close(self) -> None:
        """Close the file resource"""
        if self.file_handle:
            self.file_handle.close()
            self.is_open = False
    
    def cleanup(self) -> None:
        """Remove the file"""
        self.close()
        if os.path.exists(self.filename):
            os.remove(self.filename)

class ResourceManager:
    """Manages multiple resources with automatic cleanup"""
    def __init__(self):
        self.resources: list[FileResource] = []
    
    def create_resource(self, filename: str) -> FileResource:
        """Create and track a new resource"""
        resource = FileResource(filename)
        self.resources.append(resource)
        return resource
    
    def cleanup_all(self) -> None:
        """Clean up all resources"""
        for resource in self.resources:
            resource.cleanup()
        self.resources.clear()
    
    def __enter__(self):
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.cleanup_all()
        return False
```

Let's create a context manager to verify cleanup:

```python
# test_helpers_extended.py
from contextlib import contextmanager
import os

@contextmanager
def assert_files_cleaned_up(*filenames):
    """
    Context manager that verifies files are cleaned up after operation.
    
    Usage:
        with assert_files_cleaned_up('temp1.txt', 'temp2.txt'):
            # Create and use files
            # They should be cleaned up by the end
    """
    # Verify files don't exist before
    for filename in filenames:
        if os.path.exists(filename):
            os.remove(filename)  # Clean up from previous test
    
    # Yield control
    yield
    
    # Verify files are cleaned up after
    remaining_files = [f for f in filenames if os.path.exists(f)]
    assert not remaining_files, (
        f"Expected all files to be cleaned up, but found: {remaining_files}"
    )
```

Now test resource cleanup:

```python
# test_resources.py
import pytest
import os
from resource_manager import ResourceManager
from test_helpers_extended import assert_files_cleaned_up

def test_resource_manager_cleanup():
    """Test that ResourceManager cleans up all resources"""
    filenames = ['test1.txt', 'test2.txt', 'test3.txt']
    
    with assert_files_cleaned_up(*filenames):
        with ResourceManager() as manager:
            # Create resources
            for filename in filenames:
                resource = manager.create_resource(filename)
                resource.open()
                resource.write("test content")
            
            # Verify files exist during operation
            for filename in filenames:
                assert os.path.exists(filename)
        
        # After exiting ResourceManager context, files should be cleaned up
        # The assert_files_cleaned_up context manager will verify this

def test_manual_cleanup():
    """Test manual cleanup of resources"""
    filenames = ['manual1.txt', 'manual2.txt']
    
    with assert_files_cleaned_up(*filenames):
        manager = ResourceManager()
        
        for filename in filenames:
            resource = manager.create_resource(filename)
            resource.open()
            resource.write("test content")
        
        # Manually trigger cleanup
        manager.cleanup_all()
```

Run the tests:

```bash
pytest test_resources.py -v
```

**Output**:
```
======================== test session starts =========================
collected 2 items

test_resources.py::test_resource_manager_cleanup PASSED         [50%]
test_resources.py::test_manual_cleanup PASSED                   [100%]

========================= 2 passed in 0.08s ==========================
```

Let's see what happens when cleanup fails:

```python
# test_resources_failing.py
import pytest
import os
from resource_manager import ResourceManager
from test_helpers_extended import assert_files_cleaned_up

def test_cleanup_failure():
    """Test what happens when cleanup fails"""
    filenames = ['leak1.txt', 'leak2.txt']
    
    with assert_files_cleaned_up(*filenames):
        manager = ResourceManager()
        
        for filename in filenames:
            resource = manager.create_resource(filename)
            resource.open()
            resource.write("test content")
        
        # BUG: Forgot to call cleanup!
        # Files will still exist when context manager exits
```

Run the failing test:

```bash
pytest test_resources_failing.py -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_resources_failing.py::test_cleanup_failure FAILED          [100%]

============================== FAILURES ==============================
____________________ test_cleanup_failure ____________________________

    def test_cleanup_failure():
        filenames = ['leak1.txt', 'leak2.txt']
        
        with assert_files_cleaned_up(*filenames):
            manager = ResourceManager()
            
            for filename in filenames:
                resource = manager.create_resource(filename)
                resource.open()
                resource.write("test content")

test_helpers_extended.py:21: AssertionError
E       AssertionError: Expected all files to be cleaned up, but found: ['leak1.txt', 'leak2.txt']

test_resources_failing.py:9: 
========================= 1 failed in 0.10s ==========================
```

**Expected vs. Actual improvement**: The context manager immediately identifies which files weren't cleaned up, making resource leaks obvious.

## Iteration 15: Combining Multiple Context Managers

You can stack context managers to test multiple aspects simultaneously:

```python
# test_combined.py
import pytest
from state_machine import Order, OrderStatus, InvalidTransitionError
from test_helpers import assert_state_changes

def test_invalid_transition_preserves_state():
    """Test that failed transition doesn't change state"""
    order = Order(order_id="ORD-006")
    
    # Transition to processing first
    with assert_state_changes(order, 'status', OrderStatus.PENDING, OrderStatus.PROCESSING):
        order.transition_to(OrderStatus.PROCESSING)
    
    # Now try invalid transition - state should remain PROCESSING
    with pytest.raises(InvalidTransitionError):
        with assert_state_changes(order, 'status', OrderStatus.PROCESSING, OrderStatus.PROCESSING):
            # This will raise, but state should not change
            order.transition_to(OrderStatus.DELIVERED)  # Invalid: skips SHIPPED
```

Run the test:

```bash
pytest test_combined.py -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_combined.py::test_invalid_transition_preserves_state PASSED [100%]

========================= 1 passed in 0.07s ==========================
```

This pattern verifies both that:
1. An exception is raised (via `pytest.raises()`)
2. The state doesn't change (via `assert_state_changes()`)

## Advanced Pattern: Timing Context Manager

Let's create a context manager that verifies operations complete within a time limit:

```python
# test_helpers_timing.py
from contextlib import contextmanager
import time

@contextmanager
def assert_completes_within(seconds: float):
    """
    Context manager that verifies operation completes within time limit.
    
    Usage:
        with assert_completes_within(1.0):
            slow_operation()
    """
    start_time = time.time()
    
    yield
    
    elapsed = time.time() - start_time
    assert elapsed <= seconds, (
        f"Operation took {elapsed:.3f}s, expected <= {seconds}s"
    )
```

Use it to test performance requirements:

```python
# test_timing.py
import pytest
import time
from test_helpers_timing import assert_completes_within

def fast_operation():
    """Simulates a fast operation"""
    time.sleep(0.1)
    return "done"

def slow_operation():
    """Simulates a slow operation"""
    time.sleep(2.0)
    return "done"

def test_fast_operation_performance():
    """Test that fast operation completes quickly"""
    with assert_completes_within(0.5):
        result = fast_operation()
        assert result == "done"

def test_slow_operation_fails_timing():
    """Test that slow operation exceeds time limit"""
    with assert_completes_within(1.0):
        result = slow_operation()  # This will fail the timing assertion
        assert result == "done"
```

Run the tests:

```bash
pytest test_timing.py::test_fast_operation_performance -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_timing.py::test_fast_operation_performance PASSED          [100%]

========================= 1 passed in 0.11s ==========================
```

Run the failing test:

```bash
pytest test_timing.py::test_slow_operation_fails_timing -v
```

**Output**:
```
======================== test session starts =========================
collected 1 item

test_timing.py::test_slow_operation_fails_timing FAILED         [100%]

============================== FAILURES ==============================
_____________ test_slow_operation_fails_timing _______________________

    def test_slow_operation_fails_timing():
        with assert_completes_within(1.0):
>           result = slow_operation()

test_helpers_timing.py:18: AssertionError
E       AssertionError: Operation took 2.001s, expected <= 1.0s

test_timing.py:23: 
========================= 1 failed in 2.01s ==========================
```

## When to Create Custom Context Managers

**Create custom context managers when**:
- You have repeated setup/teardown patterns across multiple tests
- You need to verify state before and after an operation
- You want to ensure resources are cleaned up
- You need to capture and verify side effects
- You want to make test intent clearer

**Don't create custom context managers when**:
- The pattern is used in only one or two tests
- Built-in context managers (`pytest.raises()`, `pytest.warns()`) already handle your case
- The abstraction makes tests harder to understand

## Summary: The Journey from Simple to Sophisticated Assertions

We started with simple equality assertions and progressively built up to sophisticated testing patterns:

| Iteration | Challenge                      | Solution                                    | Key Technique                    |
| --------- | ------------------------------ | ------------------------------------------- | -------------------------------- |
| 0         | Basic validation               | Simple `assert` statements                  | Equality checks                  |
| 1         | Expected exceptions            | `pytest.raises()`                           | Exception capture                |
| 2         | Exception messages             | `exc_info.value`                            | Exception inspection             |
| 3         | Complex object comparison      | Pytest introspection                        | Automatic diff generation        |
| 4         | Collection comparison          | Pytest introspection                        | Index/key-specific diffs         |
| 5         | Domain context                 | Custom assertion messages                   | Second argument to `assert`      |
| 6         | Complex business logic         | Detailed custom messages                    | Multi-line explanations          |
| 7         | Exception attributes           | `exc_info.value` attribute access           | Full exception inspection        |
| 8         | Exception inheritance          | `isinstance()` checks                       | Type verification                |
| 9         | Message patterns               | `match` parameter                           | Regex matching                   |
| 10        | Warning testing                | `pytest.warns()`                            | Warning capture                  |
| 11        | Warning messages               | `warning_info` inspection                   | Warning details                  |
| 12        | Multiple warnings              | `len(warning_info)`                         | Warning counting                 |
| 13        | State changes                  | Custom context manager                      | Before/after verification        |
| 14        | Resource cleanup               | Custom cleanup context manager              | Cleanup verification             |
| 15        | Combined verification          | Stacked context managers                    | Multiple simultaneous assertions |
| Advanced  | Performance requirements       | Timing context manager                      | Time-bounded operations          |

## Decision Framework: Which Assertion Approach When?

### For Simple Value Comparisons
- **Use**: Plain `assert` with pytest introspection
- **When**: Comparing primitives, strings, numbers, simple collections
- **Example**: `assert result == expected`

### For Floating Point Comparisons
- **Use**: `pytest.approx()`
- **When**: Comparing floating point numbers or collections containing them
- **Example**: `assert result == pytest.approx(expected, rel=1e-6)`

### For Exception Testing
- **Use**: `pytest.raises(ExceptionType)`
- **When**: Verifying that code raises expected exceptions
- **Add `match`**: When you need to verify exception message format
- **Use `exc_info.value`**: When you need to inspect exception attributes
- **Example**: `with pytest.raises(ValueError, match=r"invalid.*") as exc_info:`

### For Warning Testing
- **Use**: `pytest.warns(WarningType)`
- **When**: Verifying that code issues warnings
- **Add `match`**: When you need to verify warning message format
- **Use `warning_info`**: When you need to inspect multiple warnings
- **Example**: `with pytest.warns(DeprecationWarning, match=r"deprecated") as warning_info:`

### For Complex Assertions
- **Use**: Custom assertion messages
- **When**: The assertion logic is complex or non-obvious
- **When**: You need to explain business rules
- **When**: Multiple conditions are being checked
- **Example**: `assert condition, f"Detailed explanation with {context}"`

### For State Verification
- **Use**: Custom context managers
- **When**: You need to verify before/after state
- **When**: You have repeated verification patterns
- **When**: You need to ensure cleanup happens
- **Example**: `with assert_state_changes(obj, 'attr', before, after):`

### For Resource Management
- **Use**: Custom cleanup context managers
- **When**: Testing that resources are properly released
- **When**: Verifying file/connection/handle cleanup
- **Example**: `with assert_files_cleaned_up('file1.txt', 'file2.txt'):`

## Best Practices Summary

1. **Start simple**: Use plain `assert` until you need more
2. **Let pytest introspect**: Don't add custom messages that just repeat what pytest shows
3. **Add context when needed**: Use custom messages to explain business logic
4. **Test exceptions explicitly**: Use `pytest.raises()` for expected exceptions
5. **Inspect when necessary**: Use `exc_info.value` to verify exception details
6. **Match patterns flexibly**: Use `match` parameter for regex-based verification
7. **Test warnings deliberately**: Use `pytest.warns()` for deprecation and other warnings
8. **Create abstractions judiciously**: Custom context managers for repeated patterns only
9. **Combine techniques**: Stack context managers when testing multiple aspects
10. **Make failures informative**: Every assertion should clearly explain what went wrong

The goal is not to use every technique in every test, but to choose the right tool for each situation. Simple tests should remain simple; complex scenarios deserve sophisticated assertion strategies.
