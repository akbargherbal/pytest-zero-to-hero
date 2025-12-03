# Chapter 2: Understanding Test Fundamentals

## Assertions: The Heart of Testing

## Assertions: The Heart of Testing

Every test ultimately answers one question: **Does this code behave as expected?** The mechanism that answers this question is the assertion—a statement that declares what should be true. If the assertion holds, the test passes. If it fails, the test fails, and pytest tells you exactly what went wrong.

In this section, we'll build a reference implementation that we'll refine throughout the chapter: a simple user authentication system. We'll start with basic assertions and progressively discover their power through the failures they reveal.

### The Reference Problem: User Authentication

Let's test a function that validates user credentials. This is our anchor example—realistic enough to matter, simple enough to understand immediately.

```python
# auth.py
def authenticate_user(username, password, users_db):
    """
    Authenticate a user against a database of valid credentials.
    
    Args:
        username: The username to check
        password: The password to verify
        users_db: Dictionary mapping usernames to passwords
    
    Returns:
        True if credentials are valid, False otherwise
    """
    if username not in users_db:
        return False
    return users_db[username] == password
```

### Iteration 0: The Naive First Test

Let's write our first test using the most basic assertion possible.

```python
# test_auth.py
from auth import authenticate_user

def test_valid_credentials():
    users = {"alice": "secret123", "bob": "password456"}
    result = authenticate_user("alice", "secret123", users)
    assert result
```

Run this test:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_valid_credentials PASSED                              [100%]

========================== 1 passed in 0.01s ===========================
```

Success! But this test only covers the happy path. What happens when we test invalid credentials?

```python
# test_auth.py
from auth import authenticate_user

def test_valid_credentials():
    users = {"alice": "secret123", "bob": "password456"}
    result = authenticate_user("alice", "secret123", users)
    assert result

def test_invalid_password():
    users = {"alice": "secret123", "bob": "password456"}
    result = authenticate_user("alice", "wrong_password", users)
    assert result  # This should fail!
```

Run the tests:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_valid_credentials PASSED                              [ 50%]
test_auth.py::test_invalid_password FAILED                               [100%]

=========================== FAILURES ====================================
__________________ test_invalid_password ________________________________

    def test_invalid_password():
        users = {"alice": "secret123", "bob": "password456"}
        result = authenticate_user("alice", "wrong_password", users)
>       assert result
E       assert False

test_auth.py:11: AssertionError
======================== short test summary info ========================
FAILED test_auth.py::test_invalid_password - assert False
===================== 1 failed, 1 passed in 0.03s =======================
```

### Diagnostic Analysis: Reading the Failure

**The complete output tells a story**. Let's parse it section by section.

**1. The summary line**: `FAILED test_auth.py::test_invalid_password - assert False`

What this tells us:
- Which test failed: `test_invalid_password`
- Where it lives: `test_auth.py`
- The immediate cause: `assert False`

**2. The traceback**:

```text
def test_invalid_password():
        users = {"alice": "secret123", "bob": "password456"}
        result = authenticate_user("alice", "wrong_password", users)
>       assert result
E       assert False
```

What this tells us:
- The `>` marker shows the exact line that failed
- The `E` prefix shows pytest's interpretation: `assert False`
- The assertion evaluated to `False`, which caused the failure

**3. The assertion introspection**:

```text
E       assert False
```

What this tells us:
- The variable `result` contained the value `False`
- Our assertion `assert result` expected a truthy value
- Pytest shows us the actual value that caused the failure

**Root cause identified**: We wrote `assert result` when we meant to test that the result is `False` for invalid credentials.

**Why the current approach can't solve this**: The assertion `assert result` only passes when `result` is truthy. We need a way to assert that something is `False`.

**What we need**: Explicit comparison assertions that state our expectations clearly.

### Iteration 1: Explicit Comparisons

The problem with `assert result` is ambiguity. Does it mean "result should be True" or "result should exist"? Let's be explicit about what we expect.

```python
# test_auth.py
from auth import authenticate_user

def test_valid_credentials():
    users = {"alice": "secret123", "bob": "password456"}
    result = authenticate_user("alice", "secret123", users)
    assert result == True  # Explicit: we expect True

def test_invalid_password():
    users = {"alice": "secret123", "bob": "password456"}
    result = authenticate_user("alice", "wrong_password", users)
    assert result == False  # Explicit: we expect False
```

Run the tests:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_valid_credentials PASSED                              [ 50%]
test_auth.py::test_invalid_password PASSED                               [100%]

========================== 2 passed in 0.01s ===========================
```

**Expected vs. Actual improvement**: Both tests now pass because we're explicitly stating what we expect. The assertion `result == False` passes when authentication correctly fails.

**Current limitation**: Our assertions work, but what happens when they fail? Let's intentionally break our code to see how pytest helps us debug.

### Iteration 2: Understanding Assertion Introspection

Let's introduce a bug in our authentication function to see how pytest's assertion introspection helps us diagnose problems.

```python
# auth.py (with intentional bug)
def authenticate_user(username, password, users_db):
    """
    Authenticate a user against a database of valid credentials.
    (Buggy version for demonstration)
    """
    if username not in users_db:
        return False
    # Bug: always returns True if user exists, ignoring password
    return True
```

Run the tests:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_valid_credentials PASSED                              [ 50%]
test_auth.py::test_invalid_password FAILED                               [100%]

=========================== FAILURES ====================================
__________________ test_invalid_password ________________________________

    def test_invalid_password():
        users = {"alice": "secret123", "bob": "password456"}
        result = authenticate_user("alice", "wrong_password", users)
>       assert result == False
E       assert True == False

test_auth.py:11: AssertionError
======================== short test summary info ========================
FAILED test_auth.py::test_invalid_password - assert True == False
===================== 1 failed, 1 passed in 0.03s =======================
```

### Diagnostic Analysis: Reading the Comparison Failure

**The complete output**:

```text
>       assert result == False
E       assert True == False
```

**Let's parse this**:

1. **The assertion line**: `assert result == False`
   - What we expected: `False` (invalid credentials should be rejected)
   - What we're testing: The variable `result`

2. **The introspection**: `assert True == False`
   - What we got: `True`
   - What we expected: `False`
   - The comparison: These are not equal

**Root cause identified**: The function returned `True` when we expected `False`. The bug is now obvious—our authentication function isn't checking the password.

**Key insight**: Pytest's assertion introspection shows both sides of the comparison. We don't need to add print statements or use a debugger to see what went wrong. The failure message tells us exactly what value we got versus what we expected.

Let's fix the bug:

```python
# auth.py (fixed)
def authenticate_user(username, password, users_db):
    """
    Authenticate a user against a database of valid credentials.
    """
    if username not in users_db:
        return False
    return users_db[username] == password  # Fixed: actually check password
```

Run the tests again:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_valid_credentials PASSED                              [ 50%]
test_auth.py::test_invalid_password PASSED                               [100%]

========================== 2 passed in 0.01s ===========================
```

**Expected vs. Actual improvement**: The bug is fixed, and pytest's introspection made the problem immediately visible without any debugging effort.

**Current limitation**: We're only testing boolean results. What about testing more complex values like strings, numbers, or collections?

### Iteration 3: Assertions with Complex Data Types

Let's extend our authentication system to return more information. Instead of just `True`/`False`, let's return a user profile dictionary on success.

```python
# auth.py
def authenticate_user(username, password, users_db):
    """
    Authenticate a user and return their profile.
    
    Returns:
        Dictionary with user info if valid, None if invalid
    """
    if username not in users_db:
        return None
    
    stored_password, profile = users_db[username]
    if stored_password != password:
        return None
    
    return profile
```

Now let's test this enhanced version:

```python
# test_auth.py
from auth import authenticate_user

def test_valid_credentials_returns_profile():
    users = {
        "alice": ("secret123", {"id": 1, "email": "alice@example.com", "role": "admin"}),
        "bob": ("password456", {"id": 2, "email": "bob@example.com", "role": "user"})
    }
    result = authenticate_user("alice", "secret123", users)
    expected = {"id": 1, "email": "alice@example.com", "role": "admin"}
    assert result == expected

def test_invalid_password_returns_none():
    users = {
        "alice": ("secret123", {"id": 1, "email": "alice@example.com", "role": "admin"})
    }
    result = authenticate_user("alice", "wrong_password", users)
    assert result == None
```

Run the tests:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_valid_credentials_returns_profile PASSED             [ 50%]
test_auth.py::test_invalid_password_returns_none PASSED                 [100%]

========================== 2 passed in 0.01s ===========================
```

Great! But what happens when a dictionary comparison fails? Let's introduce a bug to see pytest's introspection for complex types.

```python
# auth.py (with bug in profile)
def authenticate_user(username, password, users_db):
    """
    Authenticate a user and return their profile.
    (Buggy version - wrong email domain)
    """
    if username not in users_db:
        return None
    
    stored_password, profile = users_db[username]
    if stored_password != password:
        return None
    
    # Bug: changing email domain
    buggy_profile = profile.copy()
    buggy_profile["email"] = profile["email"].replace("@example.com", "@wrong.com")
    return buggy_profile
```

Run the tests:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_valid_credentials_returns_profile FAILED             [ 50%]
test_auth.py::test_invalid_password_returns_none PASSED                 [100%]

=========================== FAILURES ====================================
_____________ test_valid_credentials_returns_profile ____________________

    def test_valid_credentials_returns_profile():
        users = {
            "alice": ("secret123", {"id": 1, "email": "alice@example.com", "role": "admin"}),
            "bob": ("password456", {"id": 2, "email": "bob@example.com", "role": "user"})
        }
        result = authenticate_user("alice", "secret123", users)
        expected = {"id": 1, "email": "alice@example.com", "role": "admin"}
>       assert result == expected
E       AssertionError: assert {'id': 1, 'em...role': 'admin'} == {'id': 1, 'em...role': 'admin'}
E         
E         Differing items:
E         {'email': 'alice@wrong.com'} != {'email': 'alice@example.com'}
E         
E         Full diff:
E         - {'email': 'alice@example.com', 'id': 1, 'role': 'admin'}
E         ?            ^^^^^^^
E         
E         + {'email': 'alice@wrong.com', 'id': 1, 'role': 'admin'}
E         ?            ^^^^

test_auth.py:11: AssertionError
======================== short test summary info ========================
FAILED test_auth.py::test_valid_credentials_returns_profile - AssertionError
===================== 1 failed, 1 passed in 0.03s =======================
```

### Diagnostic Analysis: Reading Dictionary Comparison Failures

**The complete output shows pytest's powerful introspection for complex types**:

**1. The summary**: `assert {'id': 1, 'em...role': 'admin'} == {'id': 1, 'em...role': 'admin'}`
   - Both dictionaries look similar at first glance
   - Pytest truncates the display but shows they're not equal

**2. The detailed diff**:

```text
E         Differing items:
E         {'email': 'alice@wrong.com'} != {'email': 'alice@example.com'}
```

What this tells us:
- Pytest automatically compares dictionary keys
- It identifies exactly which key has a different value
- It shows both the actual and expected values for that key

**3. The visual diff**:

```text
E         Full diff:
E         - {'email': 'alice@example.com', 'id': 1, 'role': 'admin'}
E         ?            ^^^^^^^
E         
E         + {'email': 'alice@wrong.com', 'id': 1, 'role': 'admin'}
E         ?            ^^^^
```

What this tells us:
- The `-` line shows what we expected
- The `+` line shows what we got
- The `?` lines with `^` markers highlight the exact character differences
- The domain changed from `example` to `wrong`

**Root cause identified**: The email domain is being modified incorrectly. The visual diff makes it immediately obvious which characters differ.

**Key insight**: Pytest doesn't just say "these dictionaries are different." It shows you:
1. Which keys differ
2. What the expected vs. actual values are
3. A character-by-character diff of string differences

This level of detail eliminates guesswork. You don't need to add print statements or use a debugger—the assertion failure tells you exactly what's wrong.

Let's fix the bug and verify:

```python
# auth.py (fixed)
def authenticate_user(username, password, users_db):
    """
    Authenticate a user and return their profile.
    """
    if username not in users_db:
        return None
    
    stored_password, profile = users_db[username]
    if stored_password != password:
        return None
    
    return profile  # Fixed: return profile unchanged
```

**Expected vs. Actual improvement**: Tests pass, and we've learned that pytest provides detailed introspection for complex data structures, not just simple values.

### Iteration 4: Testing Collections and Membership

What if we need to test that a value is in a collection, or that a collection contains specific items? Let's add a feature that returns a list of user permissions.

```python
# auth.py
def get_user_permissions(username, users_db):
    """
    Get the list of permissions for a user.
    
    Returns:
        List of permission strings, or empty list if user not found
    """
    if username not in users_db:
        return []
    
    _, profile = users_db[username]
    return profile.get("permissions", [])
```

Let's test this:

```python
# test_auth.py
from auth import authenticate_user, get_user_permissions

def test_admin_has_all_permissions():
    users = {
        "alice": ("secret123", {
            "id": 1,
            "email": "alice@example.com",
            "role": "admin",
            "permissions": ["read", "write", "delete", "admin"]
        })
    }
    permissions = get_user_permissions("alice", users)
    
    # Test that specific permissions are present
    assert "read" in permissions
    assert "write" in permissions
    assert "admin" in permissions

def test_regular_user_has_limited_permissions():
    users = {
        "bob": ("password456", {
            "id": 2,
            "email": "bob@example.com",
            "role": "user",
            "permissions": ["read"]
        })
    }
    permissions = get_user_permissions("bob", users)
    
    # Test that admin permission is NOT present
    assert "admin" not in permissions
    assert "delete" not in permissions
```

Run the tests:

```bash
pytest test_auth.py::test_admin_has_all_permissions -v
pytest test_auth.py::test_regular_user_has_limited_permissions -v
```

**Output**:

```text
test_auth.py::test_admin_has_all_permissions PASSED                     [100%]
test_auth.py::test_regular_user_has_limited_permissions PASSED          [100%]

========================== 2 passed in 0.01s ===========================
```

**Key patterns demonstrated**:
- `assert item in collection` - Tests membership
- `assert item not in collection` - Tests absence

Let's see what happens when a membership assertion fails:

```python
# test_auth.py
def test_user_has_write_permission():
    users = {
        "bob": ("password456", {
            "id": 2,
            "email": "bob@example.com",
            "role": "user",
            "permissions": ["read"]  # Only has read, not write
        })
    }
    permissions = get_user_permissions("bob", users)
    assert "write" in permissions  # This will fail
```

Run the test:

```bash
pytest test_auth.py::test_user_has_write_permission -v
```

**Output**:

```text
test_auth.py::test_user_has_write_permission FAILED                     [100%]

=========================== FAILURES ====================================
________________ test_user_has_write_permission _________________________

    def test_user_has_write_permission():
        users = {
            "bob": ("password456", {
                "id": 2,
                "email": "bob@example.com",
                "role": "user",
                "permissions": ["read"]
            })
        }
        permissions = get_user_permissions("bob", users)
>       assert "write" in permissions
E       AssertionError: assert 'write' in ['read']

test_auth.py:11: AssertionError
======================== short test summary info ========================
FAILED test_auth.py::test_user_has_write_permission - AssertionError
===================== 1 failed, 1 passed in 0.03s =======================
```

### Diagnostic Analysis: Reading Membership Failures

**The introspection**: `assert 'write' in ['read']`

What this tells us:
- We're looking for the string `'write'`
- The actual collection is `['read']`
- The item we're looking for is not in the collection
- We can see the entire collection's contents

**Root cause identified**: The user only has `read` permission, not `write`. The failure message shows us both what we were looking for and what was actually in the collection.

### Iteration 5: Comparing Collections

What if we want to test that a collection contains exactly the right items, in any order? Or that it contains at least certain items?

```python
# test_auth.py
def test_admin_permissions_complete():
    users = {
        "alice": ("secret123", {
            "id": 1,
            "email": "alice@example.com",
            "role": "admin",
            "permissions": ["read", "write", "delete", "admin"]
        })
    }
    permissions = get_user_permissions("alice", users)
    expected = ["read", "write", "delete", "admin"]
    
    # Compare as sets - order doesn't matter
    assert set(permissions) == set(expected)
```

Let's introduce a bug where a permission is missing:

```python
# auth.py (with bug)
def get_user_permissions(username, users_db):
    """
    Get the list of permissions for a user.
    (Buggy version - filters out 'delete' permission)
    """
    if username not in users_db:
        return []
    
    _, profile = users_db[username]
    permissions = profile.get("permissions", [])
    # Bug: accidentally filtering out delete permission
    return [p for p in permissions if p != "delete"]
```

Run the test:

```bash
pytest test_auth.py::test_admin_permissions_complete -v
```

**Output**:

```text
test_auth.py::test_admin_permissions_complete FAILED                    [100%]

=========================== FAILURES ====================================
______________ test_admin_permissions_complete __________________________

    def test_admin_permissions_complete():
        users = {
            "alice": ("secret123", {
                "id": 1,
                "email": "alice@example.com",
                "role": "admin",
                "permissions": ["read", "write", "delete", "admin"]
            })
        }
        permissions = get_user_permissions("alice", users)
        expected = ["read", "write", "delete", "admin"]
>       assert set(permissions) == set(expected)
E       AssertionError: assert {'admin', 'read', 'write'} == {'admin', 'delete', 'read', 'write'}
E         
E         Extra items in the right set:
E         {'delete'}

test_auth.py:11: AssertionError
======================== short test summary info ========================
FAILED test_auth.py::test_admin_permissions_complete - AssertionError
===================== 1 failed, 1 passed in 0.03s =======================
```

### Diagnostic Analysis: Reading Set Comparison Failures

**The introspection**:

```text
E       AssertionError: assert {'admin', 'read', 'write'} == {'admin', 'delete', 'read', 'write'}
E         
E         Extra items in the right set:
E         {'delete'}
```

What this tells us:
- The left set (actual): `{'admin', 'read', 'write'}`
- The right set (expected): `{'admin', 'delete', 'read', 'write'}`
- Pytest identifies that `'delete'` is in the expected set but not in the actual set
- It labels this as "Extra items in the right set" (meaning items we expected but didn't get)

**Root cause identified**: The `delete` permission is missing from the actual permissions. The set comparison makes it immediately clear which items are missing or extra.

**Key insight**: When comparing collections:
- Use `==` for exact equality (order matters for lists)
- Use `set()` comparison when order doesn't matter
- Pytest shows you exactly which items are missing or extra

### Common Assertion Patterns Summary

Here are the assertion patterns we've discovered through our authentication system:

**Boolean assertions**:

```python
assert result                    # Result should be truthy
assert result == True            # Explicit: result should be True
assert result == False           # Explicit: result should be False
assert not result                # Result should be falsy
```

**Equality assertions**:

```python
assert actual == expected        # Values should be equal
assert actual != unexpected      # Values should not be equal
```

**Membership assertions**:

```python
assert item in collection        # Item should be in collection
assert item not in collection    # Item should not be in collection
```

**Collection assertions**:

```python
assert list1 == list2            # Lists equal (order matters)
assert set(list1) == set(list2)  # Same items (order doesn't matter)
assert dict1 == dict2            # Dictionaries equal
```

**Comparison assertions**:

```python
assert value > threshold         # Greater than
assert value >= minimum          # Greater than or equal
assert value < maximum           # Less than
assert value <= limit            # Less than or equal
```

### Why Plain `assert` Is Powerful

You might wonder: why does pytest use plain Python `assert` instead of special assertion methods like `assertEqual()` or `assertTrue()` from unittest?

**The answer**: Pytest rewrites your assertions at import time to provide detailed introspection. When you write:

```python
assert result == expected
```

Pytest transforms this into code that:
1. Evaluates both sides of the comparison
2. Captures the actual values
3. If the assertion fails, formats a detailed error message showing what went wrong

This means you get the simplicity of plain `assert` with the power of detailed failure messages. You don't need to remember different assertion methods—just use Python's natural comparison operators.

### When to Use Multiple Assertions

A common question: should a test have one assertion or multiple?

**General guideline**: One logical assertion per test, but that can involve multiple `assert` statements if they're testing the same concept.

**Good**: Multiple assertions testing the same concept

```python
def test_admin_has_all_permissions():
    permissions = get_user_permissions("alice", users)
    # All these assertions test the same concept: "admin has full permissions"
    assert "read" in permissions
    assert "write" in permissions
    assert "delete" in permissions
    assert "admin" in permissions
```

**Better**: Single assertion that captures the concept

```python
def test_admin_has_all_permissions():
    permissions = get_user_permissions("alice", users)
    expected = {"read", "write", "delete", "admin"}
    assert set(permissions) == expected
```

**Avoid**: Testing multiple unrelated concepts

```python
def test_authentication_and_permissions_and_email():
    # Too many unrelated concepts in one test
    result = authenticate_user("alice", "secret123", users)
    assert result is not None
    
    permissions = get_user_permissions("alice", users)
    assert "admin" in permissions
    
    email = result["email"]
    assert "@example.com" in email
```

**Why this matters**: If the first assertion fails, pytest stops executing the test. You won't see whether the other assertions would have passed or failed. Separate tests for separate concepts give you more precise failure information.

### The Journey: From Simple to Sophisticated Assertions

| Iteration | What We Tested                | Assertion Pattern                | Key Learning                                  |
| --------- | ----------------------------- | -------------------------------- | --------------------------------------------- |
| 0         | Boolean result                | `assert result`                  | Ambiguous—doesn't state expectation clearly   |
| 1         | Explicit boolean              | `assert result == True`          | Clear expectations prevent confusion          |
| 2         | Intentional failure           | Comparison with introspection    | Pytest shows both sides of failed comparisons |
| 3         | Dictionary comparison         | `assert dict1 == dict2`          | Pytest provides detailed diffs for structures |
| 4         | Collection membership         | `assert item in collection`      | Membership tests are clear and readable       |
| 5         | Set comparison                | `assert set(a) == set(b)`        | Order-independent comparison when needed      |

### Lessons Learned

**1. Be explicit**: `assert result == True` is clearer than `assert result`

**2. Trust pytest's introspection**: You don't need print statements or debuggers—the failure message shows you exactly what went wrong

**3. Use natural Python operators**: `==`, `in`, `>`, `<` all work and provide detailed failure messages

**4. Match the assertion to the concept**: Use set comparison when order doesn't matter, list comparison when it does

**5. One concept per test**: Multiple assertions are fine if they test the same logical concept, but separate tests for separate concepts give better failure isolation

## Test Functions vs. Other Functions

## Test Functions vs. Other Functions

A test function looks like a regular Python function, but pytest treats it specially. Understanding what makes a function a "test" versus a "helper" is crucial for organizing your test code effectively.

Let's continue with our authentication system and explore the distinction through practical examples.

### The Reference Problem: Expanding Authentication

Our authentication system now needs to handle multiple scenarios: valid credentials, invalid passwords, non-existent users, and edge cases like empty passwords. We'll discover when to write test functions versus helper functions.

### Iteration 0: Everything in Test Functions

Let's start by writing tests the naive way—duplicating setup code in every test.

```python
# test_auth.py
from auth import authenticate_user

def test_valid_credentials():
    # Setup
    users = {
        "alice": ("secret123", {"id": 1, "email": "alice@example.com"}),
        "bob": ("password456", {"id": 2, "email": "bob@example.com"})
    }
    
    # Test
    result = authenticate_user("alice", "secret123", users)
    assert result is not None
    assert result["id"] == 1

def test_invalid_password():
    # Setup (duplicated!)
    users = {
        "alice": ("secret123", {"id": 1, "email": "alice@example.com"}),
        "bob": ("password456", {"id": 2, "email": "bob@example.com"})
    }
    
    # Test
    result = authenticate_user("alice", "wrong_password", users)
    assert result is None

def test_nonexistent_user():
    # Setup (duplicated again!)
    users = {
        "alice": ("secret123", {"id": 1, "email": "alice@example.com"}),
        "bob": ("password456", {"id": 2, "email": "bob@example.com"})
    }
    
    # Test
    result = authenticate_user("charlie", "any_password", users)
    assert result is None
```

Run the tests:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_valid_credentials PASSED                              [ 33%]
test_auth.py::test_invalid_password PASSED                               [ 66%]
test_auth.py::test_nonexistent_user PASSED                               [100%]

========================== 3 passed in 0.01s ===========================
```

**Current limitation**: We're duplicating the same setup code in every test. This violates the DRY (Don't Repeat Yourself) principle and makes maintenance harder. If we need to add a new user to the test database, we'd have to update it in three places.

**What we need**: A way to share setup code without making pytest think it's a test.

### Iteration 1: Extracting a Helper Function

Let's extract the duplicated setup into a helper function.

```python
# test_auth.py
from auth import authenticate_user

def create_test_users():
    """Helper function to create test user database."""
    return {
        "alice": ("secret123", {"id": 1, "email": "alice@example.com"}),
        "bob": ("password456", {"id": 2, "email": "bob@example.com"})
    }

def test_valid_credentials():
    users = create_test_users()
    result = authenticate_user("alice", "secret123", users)
    assert result is not None
    assert result["id"] == 1

def test_invalid_password():
    users = create_test_users()
    result = authenticate_user("alice", "wrong_password", users)
    assert result is None

def test_nonexistent_user():
    users = create_test_users()
    result = authenticate_user("charlie", "any_password", users)
    assert result is None
```

Run the tests:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_valid_credentials PASSED                              [ 33%]
test_auth.py::test_invalid_password PASSED                               [ 66%]
test_auth.py::test_nonexistent_user PASSED                               [100%]

========================== 3 passed in 0.01s ===========================
```

**Expected vs. Actual improvement**: The tests still pass, but now we have a single source of truth for test data. Notice that pytest did NOT try to run `create_test_users()` as a test.

**Why didn't pytest run `create_test_users()` as a test?**

Pytest identifies test functions by their name. By default, pytest looks for:
- Functions that start with `test_`
- Functions inside classes that start with `Test`

Since `create_test_users()` doesn't start with `test_`, pytest ignores it. This is the key distinction: **naming determines whether a function is a test or a helper**.

### Iteration 2: What Happens If We Name It Wrong?

Let's see what happens if we accidentally name a helper function like a test.

```python
# test_auth.py
from auth import authenticate_user

def test_create_users():  # Oops! Starts with test_
    """Helper function to create test user database."""
    return {
        "alice": ("secret123", {"id": 1, "email": "alice@example.com"}),
        "bob": ("password456", {"id": 2, "email": "bob@example.com"})
    }

def test_valid_credentials():
    users = test_create_users()  # Calling our "helper"
    result = authenticate_user("alice", "secret123", users)
    assert result is not None
    assert result["id"] == 1
```

Run the tests:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_create_users PASSED                                   [ 50%]
test_auth.py::test_valid_credentials PASSED                              [100%]

========================== 2 passed in 0.02s ===========================
```

### Diagnostic Analysis: The Accidental Test

**What happened**: Pytest ran `test_create_users()` as a test!

**Why this is a problem**:
1. `test_create_users()` isn't actually testing anything—it just returns data
2. It "passes" because it doesn't raise an exception
3. This creates a false positive—a test that passes but doesn't verify any behavior
4. It pollutes your test count and can hide real issues

**The introspection**: Look at the output:

```text
test_auth.py::test_create_users PASSED                                   [ 50%]
```

Pytest collected and ran `test_create_users()` as a test. It passed because:
- The function executed without errors
- It returned a value (which pytest doesn't care about)
- There were no assertions to fail

**Root cause identified**: The function name starts with `test_`, so pytest treats it as a test.

**What we need**: Clear naming conventions that distinguish tests from helpers.

### Iteration 3: Proper Naming Conventions

Let's establish clear naming patterns for different types of functions.

```python
# test_auth.py
from auth import authenticate_user

# Helper functions - don't start with test_
def create_test_users():
    """Create a standard test user database."""
    return {
        "alice": ("secret123", {"id": 1, "email": "alice@example.com"}),
        "bob": ("password456", {"id": 2, "email": "bob@example.com"})
    }

def create_admin_user():
    """Create a user with admin privileges."""
    return ("admin123", {
        "id": 999,
        "email": "admin@example.com",
        "role": "admin",
        "permissions": ["read", "write", "delete", "admin"]
    })

def assert_valid_profile(profile):
    """Helper to verify a profile has required fields."""
    assert "id" in profile
    assert "email" in profile
    assert "@" in profile["email"]

# Test functions - start with test_
def test_valid_credentials():
    users = create_test_users()
    result = authenticate_user("alice", "secret123", users)
    assert result is not None
    assert_valid_profile(result)

def test_invalid_password():
    users = create_test_users()
    result = authenticate_user("alice", "wrong_password", users)
    assert result is None

def test_admin_authentication():
    users = {"admin": create_admin_user()}
    result = authenticate_user("admin", "admin123", users)
    assert result is not None
    assert result["role"] == "admin"
```

Run the tests:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_valid_credentials PASSED                              [ 33%]
test_auth.py::test_invalid_password PASSED                               [ 66%]
test_auth.py::test_admin_authentication PASSED                           [100%]

========================== 3 passed in 0.01s ===========================
```

**Expected vs. Actual improvement**: Only the three actual test functions ran. The helper functions were ignored by pytest but used by the tests.

**Key patterns demonstrated**:
- `create_*()` - Functions that build test data
- `assert_*()` - Functions that perform common assertions
- `test_*()` - Actual test functions that pytest runs

### Iteration 4: Understanding Test Function Signatures

Test functions have special requirements. Let's explore what makes a valid test function signature.

```python
# test_auth.py

# Valid test function - no parameters
def test_simple():
    assert True

# Valid test function - will learn about fixtures in Chapter 4
# For now, just know that pytest can inject parameters
def test_with_fixture(tmp_path):  # tmp_path is a built-in pytest fixture
    # tmp_path is automatically provided by pytest
    test_file = tmp_path / "test.txt"
    test_file.write_text("data")
    assert test_file.read_text() == "data"

# Invalid test function - requires a parameter that pytest doesn't know about
def test_with_required_param(username):
    users = create_test_users()
    result = authenticate_user(username, "secret123", users)
    assert result is not None
```

Run the tests:

```bash
pytest test_auth.py::test_with_required_param -v
```

**Output**:

```text
test_auth.py::test_with_required_param ERROR                            [100%]

=========================== ERRORS =======================================
_____________ ERROR at setup of test_with_required_param ________________

file test_auth.py, line 15
  def test_with_required_param(username):
E       fixture 'username' not found
>       available fixtures: cache, capfd, capfdbinary, caplog, capsys, capsysbinary, doctest_namespace, monkeypatch, pytestconfig, record_property, record_testsuite_property, record_xml_attribute, recwarn, tmp_path, tmp_path_factory, tmpdir, tmpdir_factory
>       use 'pytest --fixtures [testpath]' for help on them.

test_auth.py:15
======================== 1 error in 0.01s ================================
```

### Diagnostic Analysis: Understanding Test Function Parameters

**The error**: `fixture 'username' not found`

**What this tells us**:
1. Pytest tried to run the test
2. It saw the parameter `username`
3. It looked for a fixture named `username` (we'll learn about fixtures in Chapter 4)
4. It couldn't find one, so it failed during setup

**Key insight**: Test functions can only have parameters that pytest knows how to provide. These are called "fixtures" and we'll cover them in depth in Chapter 4. For now, remember:
- Test functions with no parameters always work
- Test functions with parameters must use pytest fixtures
- You can't just add arbitrary parameters to test functions

### Iteration 5: Helper Functions That Return vs. Assert

We've seen two types of helper functions: those that return data and those that perform assertions. Let's understand when to use each.

```python
# test_auth.py
from auth import authenticate_user

# Helper that RETURNS data
def create_test_users():
    """Returns test data for use in tests."""
    return {
        "alice": ("secret123", {"id": 1, "email": "alice@example.com"}),
        "bob": ("password456", {"id": 2, "email": "bob@example.com"})
    }

# Helper that ASSERTS (performs verification)
def assert_valid_profile(profile):
    """Verifies that a profile has all required fields."""
    assert profile is not None, "Profile should not be None"
    assert "id" in profile, "Profile must have an id"
    assert "email" in profile, "Profile must have an email"
    assert "@" in profile["email"], "Email must be valid"

# Helper that RETURNS a computed value
def get_user_count(users_db):
    """Returns the number of users in the database."""
    return len(users_db)

# Test using all three types of helpers
def test_authentication_with_helpers():
    # Use data-returning helper
    users = create_test_users()
    
    # Use computation helper
    user_count = get_user_count(users)
    assert user_count == 2
    
    # Perform authentication
    result = authenticate_user("alice", "secret123", users)
    
    # Use assertion helper
    assert_valid_profile(result)
```

Run the test:

```bash
pytest test_auth.py::test_authentication_with_helpers -v
```

**Output**:

```text
test_auth.py::test_authentication_with_helpers PASSED                   [100%]

========================== 1 passed in 0.01s ===========================
```

Now let's see what happens when an assertion helper fails:

```python
# test_auth.py
def test_invalid_profile():
    # Create an invalid profile (missing email)
    invalid_profile = {"id": 1}
    
    # This will fail in the helper
    assert_valid_profile(invalid_profile)
```

Run the test:

```bash
pytest test_auth.py::test_invalid_profile -v
```

**Output**:

```text
test_auth.py::test_invalid_profile FAILED                               [100%]

=========================== FAILURES ====================================
__________________ test_invalid_profile _________________________________

    def test_invalid_profile():
        invalid_profile = {"id": 1}
>       assert_valid_profile(invalid_profile)

test_auth.py:25: 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _

profile = {'id': 1}

    def assert_valid_profile(profile):
        """Verifies that a profile has all required fields."""
        assert profile is not None, "Profile should not be None"
>       assert "email" in profile, "Profile must have an email"
E       AssertionError: Profile must have an email

test_auth.py:18: AssertionError
======================== short test summary info ========================
FAILED test_auth.py::test_invalid_profile - AssertionError: Profile must have an email
===================== 1 failed, 1 passed in 0.03s =======================
```

### Diagnostic Analysis: Tracing Through Helper Functions

**The traceback shows two levels**:

**1. The test function**:

```text
def test_invalid_profile():
        invalid_profile = {"id": 1}
>       assert_valid_profile(invalid_profile)
```

This shows where in the test we called the helper.

**2. The helper function**:

```text
def assert_valid_profile(profile):
        """Verifies that a profile has all required fields."""
        assert profile is not None, "Profile should not be None"
>       assert "email" in profile, "Profile must have an email"
E       AssertionError: Profile must have an email
```

This shows which specific assertion in the helper failed.

**Key insight**: Pytest traces through your helper functions and shows you exactly where the failure occurred. The custom message "Profile must have an email" makes it immediately clear what went wrong.

### When to Use Helper Functions vs. Test Functions

**Use a helper function when**:
- You need to create test data (return values)
- You need to perform common setup operations
- You want to reuse assertion logic across multiple tests
- You need to compute values used in tests

**Use a test function when**:
- You're verifying a specific behavior
- You want pytest to discover and run it
- You want it to appear in test reports
- You want it to count toward your test coverage

### Common Patterns for Helper Functions

**Pattern 1: Data builders**

```python
def create_user(username="alice", password="secret123", **kwargs):
    """Flexible user builder with defaults."""
    profile = {
        "id": kwargs.get("id", 1),
        "email": kwargs.get("email", f"{username}@example.com"),
        "role": kwargs.get("role", "user")
    }
    return (password, profile)

# Usage in tests
def test_with_custom_user():
    users = {"admin": create_user("admin", role="admin", id=999)}
    result = authenticate_user("admin", "secret123", users)
    assert result["role"] == "admin"
```

**Pattern 2: Assertion helpers**

```python
def assert_authentication_failed(username, password, users):
    """Verify that authentication fails for given credentials."""
    result = authenticate_user(username, password, users)
    assert result is None, f"Expected authentication to fail for {username}"

# Usage in tests
def test_multiple_invalid_scenarios():
    users = create_test_users()
    assert_authentication_failed("alice", "wrong", users)
    assert_authentication_failed("nonexistent", "any", users)
    assert_authentication_failed("alice", "", users)
```

**Pattern 3: Setup helpers**

```python
def setup_database_with_users(user_count=10):
    """Create a database with multiple test users."""
    users = {}
    for i in range(user_count):
        username = f"user{i}"
        users[username] = create_user(username, id=i)
    return users

# Usage in tests
def test_large_user_database():
    users = setup_database_with_users(100)
    result = authenticate_user("user50", "secret123", users)
    assert result["id"] == 50
```

### The Journey: From Duplication to Organization

| Iteration | Approach                      | Problem                                | Solution                                |
| --------- | ----------------------------- | -------------------------------------- | --------------------------------------- |
| 0         | Duplicate setup in each test  | Violates DRY, hard to maintain         | Extract to helper function              |
| 1         | Helper function               | Works, but naming is arbitrary         | Establish naming conventions            |
| 2         | Accidentally named like test  | Helper runs as test, creates confusion | Use `test_` prefix only for tests       |
| 3         | Proper naming conventions     | Clear distinction between types        | `create_*`, `assert_*`, `test_*`        |
| 4         | Test function parameters      | Can't add arbitrary parameters         | Use fixtures (Chapter 4) or no params   |
| 5         | Different helper types        | When to return vs. assert?             | Return data, assert for verification    |

### Lessons Learned

**1. Naming determines behavior**: Functions starting with `test_` are tests; everything else is a helper

**2. Test functions are special**: They can only have parameters that pytest provides (fixtures)

**3. Helper functions are normal Python**: They can have any parameters, return values, and be called like regular functions

**4. Use descriptive prefixes**: `create_*` for builders, `assert_*` for verification helpers, `setup_*` for initialization

**5. Helpers improve readability**: Well-named helpers make tests read like documentation

**6. Assertion helpers provide context**: Custom error messages in helpers make failures clearer

**7. Don't over-abstract**: If a helper is only used once, it might not be worth extracting

## Test Discovery: How Pytest Finds Your Tests

## Test Discovery: How Pytest Finds Your Tests

When you run `pytest`, it doesn't just execute every Python file in your project. It follows specific rules to discover which files contain tests, which functions are tests, and which classes contain test methods. Understanding these rules is essential for organizing your test suite effectively.

Let's explore test discovery through our authentication system, progressively building a realistic project structure.

### The Reference Problem: Organizing a Growing Test Suite

Our authentication system is expanding. We now have:
- User authentication
- Permission management
- Session handling
- Password validation

We need to organize these tests so pytest can find them all automatically.

### Iteration 0: A Single Test File

Let's start with the simplest case—a single test file in the same directory as our code.

```bash
# Project structure
my_project/
├── auth.py
└── test_auth.py
```

```python
# test_auth.py
from auth import authenticate_user

def test_valid_credentials():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "secret123", users)
    assert result is not None
```

Run pytest from the project directory:

```bash
cd my_project
pytest
```

**Output**:

```text
========================== test session starts ==========================
collected 1 item

test_auth.py .                                                         [100%]

========================== 1 passed in 0.01s ===========================
```

**What happened**: Pytest automatically found `test_auth.py` and ran the test function inside it.

**Current limitation**: As our project grows, mixing test files with source code becomes messy. We need a better organization strategy.

### Iteration 1: Creating a Tests Directory

Let's separate tests from source code by creating a dedicated `tests/` directory.

```bash
# New project structure
my_project/
├── auth.py
└── tests/
    └── test_auth.py
```

```python
# tests/test_auth.py
import sys
sys.path.insert(0, '..')  # Temporary hack to import auth

from auth import authenticate_user

def test_valid_credentials():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "secret123", users)
    assert result is not None
```

Run pytest from the project root:

```bash
cd my_project
pytest
```

**Output**:

```text
========================== test session starts ==========================
collected 1 item

tests/test_auth.py .                                                   [100%]

========================== 1 passed in 0.01s ===========================
```

**Expected vs. Actual improvement**: Pytest found the test in the `tests/` directory automatically. But notice we had to add a hack (`sys.path.insert`) to import our code. This is not ideal.

**Current limitation**: The import hack is fragile and doesn't scale. We need a proper project structure.

### Iteration 2: Proper Package Structure

Let's create a proper Python package structure.

```bash
# Proper project structure
my_project/
├── src/
│   └── myapp/
│       ├── __init__.py
│       └── auth.py
└── tests/
    ├── __init__.py
    └── test_auth.py
```

```python
# src/myapp/__init__.py
# Empty file to make myapp a package

# src/myapp/auth.py
def authenticate_user(username, password, users_db):
    if username not in users_db:
        return None
    stored_password, profile = users_db[username]
    if stored_password != password:
        return None
    return profile
```

```python
# tests/__init__.py
# Empty file to make tests a package

# tests/test_auth.py
from myapp.auth import authenticate_user

def test_valid_credentials():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "secret123", users)
    assert result is not None
```

Install the package in development mode:

```bash
cd my_project
pip install -e src/
```

Run pytest:

```bash
pytest
```

**Output**:

```text
========================== test session starts ==========================
collected 1 item

tests/test_auth.py .                                                   [100%]

========================== 1 passed in 0.01s ===========================
```

**Expected vs. Actual improvement**: Now we can import our code cleanly without path hacks. The package structure is professional and maintainable.

**Current limitation**: We only have one test file. As we add more tests, we need to understand exactly how pytest discovers them.

### Iteration 3: Understanding Discovery Rules

Let's create multiple test files and see which ones pytest finds.

```bash
# Expanded project structure
my_project/
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── auth.py
│       └── permissions.py
└── tests/
    ├── __init__.py
    ├── test_auth.py
    ├── test_permissions.py
    ├── auth_test.py          # Wrong naming!
    └── test_helpers.py
```

```python
# tests/test_auth.py
from myapp.auth import authenticate_user

def test_valid_credentials():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "secret123", users)
    assert result is not None

def test_invalid_password():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "wrong", users)
    assert result is None
```

```python
# tests/test_permissions.py
from myapp.permissions import check_permission

def test_admin_has_all_permissions():
    assert check_permission("admin", "delete") == True

def test_user_has_limited_permissions():
    assert check_permission("user", "delete") == False
```

```python
# tests/auth_test.py (wrong naming!)
from myapp.auth import authenticate_user

def test_something():
    # This test won't be discovered!
    assert True
```

```python
# tests/test_helpers.py
def create_test_user():
    # Not a test - doesn't start with test_
    return {"alice": ("secret123", {"id": 1})}

def test_user_creation():
    # This IS a test
    user = create_test_user()
    assert "alice" in user
```

Run pytest with verbose output to see what it discovers:

```bash
pytest --collect-only
```

**Output**:

```text
========================== test session starts ==========================
collected 5 items

<Module tests/test_auth.py>
  <Function test_valid_credentials>
  <Function test_invalid_password>
<Module tests/test_permissions.py>
  <Function test_admin_has_all_permissions>
  <Function test_user_has_limited_permissions>
<Module tests/test_helpers.py>
  <Function test_user_creation>

==================== 5 tests collected in 0.01s ========================
```

### Diagnostic Analysis: What Pytest Found (and Didn't Find)

**What pytest discovered**:
1. `tests/test_auth.py` - ✓ Starts with `test_`
   - `test_valid_credentials` - ✓ Starts with `test_`
   - `test_invalid_password` - ✓ Starts with `test_`

2. `tests/test_permissions.py` - ✓ Starts with `test_`
   - `test_admin_has_all_permissions` - ✓ Starts with `test_`
   - `test_user_has_limited_permissions` - ✓ Starts with `test_`

3. `tests/test_helpers.py` - ✓ Starts with `test_`
   - `create_test_user` - ✗ Doesn't start with `test_` (correctly ignored as helper)
   - `test_user_creation` - ✓ Starts with `test_`

**What pytest did NOT discover**:
- `tests/auth_test.py` - ✗ Doesn't start with `test_` or end with `_test.py`

**Root cause identified**: Pytest follows specific naming conventions. Files must start with `test_` or end with `_test.py`. Functions must start with `test_`.

**Key insight**: The `--collect-only` flag is invaluable for debugging test discovery. It shows you exactly what pytest found without running any tests.

### Iteration 4: Configuring Discovery Patterns

What if you have a different naming convention? You can configure pytest to recognize your patterns.

```bash
# Project structure with custom naming
my_project/
├── src/
│   └── myapp/
│       ├── __init__.py
│       └── auth.py
├── tests/
│   ├── __init__.py
│   ├── test_auth.py
│   └── auth_test.py
└── pytest.ini
```

```ini
# pytest.ini
[pytest]
python_files = test_*.py *_test.py
python_classes = Test* *Tests
python_functions = test_* check_*
```

Now run collection again:

```bash
pytest --collect-only
```

**Output**:

```text
========================== test session starts ==========================
collected 6 items

<Module tests/test_auth.py>
  <Function test_valid_credentials>
  <Function test_invalid_password>
<Module tests/auth_test.py>
  <Function test_something>
<Module tests/test_permissions.py>
  <Function test_admin_has_all_permissions>
  <Function test_user_has_limited_permissions>
<Module tests/test_helpers.py>
  <Function test_user_creation>

==================== 6 tests collected in 0.01s ========================
```

**Expected vs. Actual improvement**: Now `auth_test.py` is discovered because we configured pytest to recognize the `*_test.py` pattern.

**Configuration options explained**:
- `python_files`: Patterns for test file names
- `python_classes`: Patterns for test class names (we'll cover classes in Chapter 10)
- `python_functions`: Patterns for test function names

**Current limitation**: We're discovering tests, but what about organizing them into subdirectories?

### Iteration 5: Organizing Tests in Subdirectories

As your test suite grows, you'll want to organize tests into subdirectories that mirror your source code structure.

```bash
# Organized project structure
my_project/
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── auth/
│       │   ├── __init__.py
│       │   ├── authentication.py
│       │   └── permissions.py
│       └── sessions/
│           ├── __init__.py
│           └── manager.py
└── tests/
    ├── __init__.py
    ├── auth/
    │   ├── __init__.py
    │   ├── test_authentication.py
    │   └── test_permissions.py
    └── sessions/
        ├── __init__.py
        └── test_manager.py
```

```python
# tests/auth/test_authentication.py
from myapp.auth.authentication import authenticate_user

def test_valid_credentials():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "secret123", users)
    assert result is not None
```

```python
# tests/auth/test_permissions.py
from myapp.auth.permissions import check_permission

def test_admin_permissions():
    assert check_permission("admin", "delete") == True
```

```python
# tests/sessions/test_manager.py
from myapp.sessions.manager import create_session

def test_session_creation():
    session = create_session("alice")
    assert session["user"] == "alice"
```

Run pytest with verbose output:

```bash
pytest -v
```

**Output**:

```text
========================== test session starts ==========================
collected 3 items

tests/auth/test_authentication.py::test_valid_credentials PASSED       [ 33%]
tests/auth/test_permissions.py::test_admin_permissions PASSED          [ 66%]
tests/sessions/test_manager.py::test_session_creation PASSED           [100%]

========================== 3 passed in 0.02s ===========================
```

**Expected vs. Actual improvement**: Pytest recursively discovers tests in subdirectories. The test paths in the output show the directory structure, making it easy to locate tests.

**Key insight**: Pytest automatically traverses subdirectories looking for test files. You don't need to configure anything—just follow the naming conventions.

### Iteration 6: Understanding What Pytest Ignores

Pytest doesn't search everywhere. Let's see what it ignores by default.

```bash
# Project structure with various directories
my_project/
├── src/
│   └── myapp/
│       └── auth.py
├── tests/
│   └── test_auth.py
├── .git/
│   └── test_something.py      # Ignored - hidden directory
├── node_modules/
│   └── test_something.py      # Ignored - common dependency dir
├── venv/
│   └── test_something.py      # Ignored - virtual environment
├── build/
│   └── test_something.py      # Ignored - build directory
└── __pycache__/
    └── test_something.py      # Ignored - Python cache
```

Run pytest with verbose collection:

```bash
pytest --collect-only -v
```

**Output**:

```text
========================== test session starts ==========================
collected 1 item

<Module tests/test_auth.py>
  <Function test_valid_credentials>

==================== 1 test collected in 0.01s =========================
```

**What pytest ignored**:
- `.git/` - Hidden directories (start with `.`)
- `node_modules/` - Common dependency directories
- `venv/` - Virtual environment directories
- `build/` - Build output directories
- `__pycache__/` - Python cache directories

**Why this matters**: Pytest is smart about not searching in directories that typically don't contain tests. This makes discovery fast even in large projects.

**You can customize this** in `pytest.ini`:

```ini
# pytest.ini
[pytest]
norecursedirs = .git node_modules venv build dist *.egg-info
```

### Common Discovery Patterns Summary

**Default file patterns**:
- `test_*.py` - Files starting with `test_`
- `*_test.py` - Files ending with `_test.py`

**Default function patterns**:
- `test_*` - Functions starting with `test_`

**Default class patterns** (covered in Chapter 10):
- `Test*` - Classes starting with `Test`

**Directories pytest searches**:
- Current directory and all subdirectories
- Excludes hidden directories (`.something`)
- Excludes common dependency/build directories

**Directories pytest ignores by default**:
- `.git`, `.tox`, `.nox`, `.pytest_cache`
- `node_modules`, `venv`, `env`, `virtualenv`
- `build`, `dist`, `*.egg-info`
- `__pycache__`

### Debugging Discovery Issues

When pytest doesn't find your tests, use these commands:

**See what pytest collected**:

```bash
pytest --collect-only
```

**See verbose collection information**:

```bash
pytest --collect-only -v
```

**See why pytest is ignoring certain paths**:

```bash
pytest --collect-only -v --debug
```

**Check your configuration**:

```bash
pytest --version -v
```

This shows where pytest is reading configuration from.

### The Journey: From Single File to Organized Suite

| Iteration | Structure                     | Discovery Method                  | Key Learning                                |
| --------- | ----------------------------- | --------------------------------- | ------------------------------------------- |
| 0         | Single test file              | Automatic in current directory    | Pytest finds `test_*.py` automatically      |
| 1         | Tests directory               | Automatic recursive search        | Pytest searches subdirectories              |
| 2         | Package structure             | Import from installed package     | Proper structure enables clean imports      |
| 3         | Multiple test files           | Pattern matching                  | File and function names must match patterns |
| 4         | Custom naming                 | Configuration in pytest.ini       | Discovery patterns are configurable         |
| 5         | Subdirectory organization     | Recursive discovery               | Mirror source structure in tests            |
| 6         | Understanding ignored paths   | Default exclusion patterns        | Pytest skips common non-test directories    |

### Lessons Learned

**1. Naming is discovery**: Files and functions must match pytest's patterns to be discovered

**2. Use `--collect-only`**: This flag is your best friend for debugging discovery issues

**3. Mirror your source structure**: Organize tests to match your source code layout

**4. Leverage automatic discovery**: Don't fight pytest's conventions—embrace them

**5. Configure when needed**: Use `pytest.ini` to customize discovery patterns for your project

**6. Pytest is smart about exclusions**: It automatically skips directories that typically don't contain tests

**7. Package structure matters**: Proper Python packaging makes imports clean and reliable

## Naming Conventions That Matter

## Naming Conventions That Matter

Test names are not just labels—they're documentation. A well-named test tells you what it tests, what conditions it assumes, and what outcome it expects. A poorly named test forces you to read the code to understand its purpose.

In this section, we'll explore naming conventions through our authentication system, discovering how names impact test readability, maintainability, and debugging.

### The Reference Problem: Naming Tests for Clarity

Our authentication system has grown complex. We need to test:
- Valid credentials with various user types
- Invalid credentials in different ways
- Edge cases like empty passwords, special characters
- Error conditions like missing users, database failures

Let's discover how naming conventions help us organize and understand these tests.

### Iteration 0: Generic Names

Let's start with poorly named tests and see what problems emerge.

```python
# test_auth.py
from myapp.auth import authenticate_user

def test_auth():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "secret123", users)
    assert result is not None

def test_auth2():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "wrong", users)
    assert result is None

def test_auth3():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("bob", "any", users)
    assert result is None
```

Run the tests:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_auth PASSED                                          [ 33%]
test_auth.py::test_auth2 PASSED                                         [ 66%]
test_auth.py::test_auth3 PASSED                                         [100%]

========================== 3 passed in 0.01s ===========================
```

**Current limitation**: The test names tell us nothing. What does `test_auth2` test? What's the difference between `test_auth` and `test_auth3`? We have to read the code to understand.

Now let's intentionally break one test to see how the name affects debugging:

```python
# test_auth.py (with bug)
def test_auth2():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "wrong", users)
    assert result is not None  # Bug: should be None
```

Run the tests:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_auth PASSED                                          [ 33%]
test_auth.py::test_auth2 FAILED                                         [ 66%]
test_auth.py::test_auth3 PASSED                                         [100%]

=========================== FAILURES ====================================
__________________________ test_auth2 ___________________________________

    def test_auth2():
        users = {"alice": ("secret123", {"id": 1})}
        result = authenticate_user("alice", "wrong", users)
>       assert result is not None
E       assert None is not None

test_auth.py:11: AssertionError
======================== short test summary info ========================
FAILED test_auth.py::test_auth2 - assert None is not None
===================== 1 failed, 2 passed in 0.03s =======================
```

### Diagnostic Analysis: The Cost of Poor Names

**The failure message**: `FAILED test_auth.py::test_auth2 - assert None is not None`

**What we know**:
- A test named `test_auth2` failed
- The assertion `assert None is not None` failed

**What we don't know**:
- What scenario was being tested?
- What was the expected behavior?
- Is this testing valid or invalid credentials?
- What user was involved?

**Root cause of confusion**: The name `test_auth2` provides zero context. We must read the test code to understand what failed.

**What we need**: Descriptive names that document the test's purpose.

### Iteration 1: Descriptive Names

Let's rename our tests to describe what they actually test.

```python
# test_auth.py
from myapp.auth import authenticate_user

def test_authenticate_user_with_valid_credentials():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "secret123", users)
    assert result is not None

def test_authenticate_user_with_invalid_password():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "wrong", users)
    assert result is None

def test_authenticate_user_with_nonexistent_username():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("bob", "any", users)
    assert result is None
```

Run the tests:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_authenticate_user_with_valid_credentials PASSED      [ 33%]
test_auth.py::test_authenticate_user_with_invalid_password PASSED       [ 66%]
test_auth.py::test_authenticate_user_with_nonexistent_username PASSED   [100%]

========================== 3 passed in 0.01s ===========================
```

**Expected vs. Actual improvement**: Now the test names tell a story. We can see at a glance:
- What function is being tested (`authenticate_user`)
- What scenario is being tested (`with_valid_credentials`, `with_invalid_password`)
- What the expected outcome is (implied by the scenario)

Let's introduce the same bug and see how the name helps:

```python
# test_auth.py (with bug)
def test_authenticate_user_with_invalid_password():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "wrong", users)
    assert result is not None  # Bug: should be None
```

Run the tests:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_authenticate_user_with_valid_credentials PASSED      [ 33%]
test_auth.py::test_authenticate_user_with_invalid_password FAILED       [ 66%]
test_auth.py::test_authenticate_user_with_nonexistent_username PASSED   [100%]

=========================== FAILURES ====================================
__________ test_authenticate_user_with_invalid_password _________________

    def test_authenticate_user_with_invalid_password():
        users = {"alice": ("secret123", {"id": 1})}
        result = authenticate_user("alice", "wrong", users)
>       assert result is not None
E       assert None is not None

test_auth.py:11: AssertionError
======================== short test summary info ========================
FAILED test_auth.py::test_authenticate_user_with_invalid_password
===================== 1 failed, 2 passed in 0.03s =======================
```

### Diagnostic Analysis: The Value of Descriptive Names

**The failure message**: `FAILED test_auth.py::test_authenticate_user_with_invalid_password`

**What we now know immediately**:
- The test is about authenticating a user
- The scenario involves an invalid password
- The test failed, meaning authentication didn't behave as expected for invalid passwords

**Root cause is clearer**: The name tells us this test is specifically about invalid passwords. We can immediately hypothesize that either:
1. The test expectation is wrong, or
2. The authentication function isn't properly rejecting invalid passwords

**Key insight**: A descriptive name acts as inline documentation. When a test fails in CI, you can often diagnose the issue from the name alone, without reading the code.

### Iteration 2: Naming Patterns for Different Scenarios

Let's expand our test suite to cover more scenarios and establish naming patterns.

```python
# test_auth.py
from myapp.auth import authenticate_user

# Pattern: test_<function>_<scenario>_<expected_outcome>

# Valid scenarios
def test_authenticate_user_with_valid_credentials_returns_profile():
    users = {"alice": ("secret123", {"id": 1, "email": "alice@example.com"})}
    result = authenticate_user("alice", "secret123", users)
    assert result == {"id": 1, "email": "alice@example.com"}

def test_authenticate_user_with_admin_credentials_returns_admin_profile():
    users = {"admin": ("admin123", {"id": 999, "role": "admin"})}
    result = authenticate_user("admin", "admin123", users)
    assert result["role"] == "admin"

# Invalid scenarios
def test_authenticate_user_with_invalid_password_returns_none():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "wrong", users)
    assert result is None

def test_authenticate_user_with_nonexistent_user_returns_none():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("bob", "any", users)
    assert result is None

# Edge cases
def test_authenticate_user_with_empty_password_returns_none():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "", users)
    assert result is None

def test_authenticate_user_with_empty_username_returns_none():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("", "secret123", users)
    assert result is None

def test_authenticate_user_with_case_sensitive_password_returns_none():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "SECRET123", users)
    assert result is None
```

Run the tests:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_authenticate_user_with_valid_credentials_returns_profile PASSED [ 14%]
test_auth.py::test_authenticate_user_with_admin_credentials_returns_admin_profile PASSED [ 28%]
test_auth.py::test_authenticate_user_with_invalid_password_returns_none PASSED [ 42%]
test_auth.py::test_authenticate_user_with_nonexistent_user_returns_none PASSED [ 57%]
test_auth.py::test_authenticate_user_with_empty_password_returns_none PASSED [ 71%]
test_auth.py::test_authenticate_user_with_empty_username_returns_none PASSED [ 85%]
test_auth.py::test_authenticate_user_with_case_sensitive_password_returns_none PASSED [100%]

========================== 7 passed in 0.02s ===========================
```

**Expected vs. Actual improvement**: The test names now form a specification. Reading the test list tells you:
- What the function does
- What scenarios it handles
- What the expected behavior is in each case

**Naming pattern established**:
```
test_<function_name>_<scenario_description>_<expected_outcome>
```

**Current limitation**: These names are getting long. Is there a balance between descriptiveness and brevity?

### Iteration 3: Balancing Descriptiveness and Brevity

Long names are good for clarity, but they can become unwieldy. Let's explore when to abbreviate and when to be verbose.

```python
# test_auth.py

# Too verbose - the outcome is implied by the scenario
def test_authenticate_user_with_valid_credentials_returns_profile_successfully():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "secret123", users)
    assert result is not None

# Better - outcome is clear from context
def test_authenticate_user_with_valid_credentials():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "secret123", users)
    assert result is not None

# Too brief - what's being tested?
def test_invalid():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "wrong", users)
    assert result is None

# Better - specific about what's invalid
def test_authenticate_user_rejects_invalid_password():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "wrong", users)
    assert result is None

# Good balance - clear scenario, implied outcome
def test_authenticate_user_with_empty_password():
    users = {"alice": ("secret123", {"id": 1})}
    result = authenticate_user("alice", "", users)
    assert result is None
```

**Guidelines for balancing length**:

1. **Always include the function name** (unless it's obvious from the file name)
2. **Always include the scenario** (what conditions are being tested)
3. **Include the outcome** only if it's not obvious from the scenario
4. **Use action verbs** for clarity: `rejects`, `accepts`, `returns`, `raises`

**Examples of good balance**:

```python
# Clear and concise
def test_authenticate_user_with_valid_credentials():
    pass

def test_authenticate_user_rejects_invalid_password():
    pass

def test_authenticate_user_rejects_nonexistent_user():
    pass

# When outcome needs emphasis
def test_authenticate_user_returns_full_profile():
    pass

def test_authenticate_user_raises_error_on_database_failure():
    pass
```

### Iteration 4: Naming for Different Test Types

Different types of tests benefit from different naming patterns. Let's explore patterns for various test categories.

**Unit tests** (testing a single function):

```python
# Pattern: test_<function>_<scenario>
def test_authenticate_user_with_valid_credentials():
    pass

def test_authenticate_user_rejects_invalid_password():
    pass
```

**Integration tests** (testing multiple components):

```python
# Pattern: test_<workflow>_<scenario>
def test_login_workflow_with_valid_credentials():
    # Tests authentication + session creation + redirect
    pass

def test_login_workflow_with_invalid_credentials():
    # Tests authentication failure + error message + no session
    pass
```

**Edge case tests**:

```python
# Pattern: test_<function>_<edge_case>
def test_authenticate_user_with_unicode_password():
    pass

def test_authenticate_user_with_very_long_username():
    pass

def test_authenticate_user_with_sql_injection_attempt():
    pass
```

**Error condition tests**:

```python
# Pattern: test_<function>_raises_<error>_when_<condition>
def test_authenticate_user_raises_value_error_when_username_is_none():
    pass

def test_authenticate_user_raises_database_error_when_connection_fails():
    pass
```

**Performance tests**:

```python
# Pattern: test_<function>_<performance_aspect>
def test_authenticate_user_completes_within_100ms():
    pass

def test_authenticate_user_handles_1000_concurrent_requests():
    pass
```

### Iteration 5: Using Test Names for Documentation

Test names can serve as living documentation. Let's see how well-named tests document behavior.

```python
# test_auth.py - Authentication Behavior Documentation

# Basic authentication
def test_authenticate_user_with_valid_credentials():
    """Valid username and password return user profile."""
    pass

def test_authenticate_user_rejects_invalid_password():
    """Invalid password returns None."""
    pass

def test_authenticate_user_rejects_nonexistent_user():
    """Non-existent username returns None."""
    pass

# Password requirements
def test_authenticate_user_rejects_empty_password():
    """Empty password is not allowed."""
    pass

def test_authenticate_user_accepts_password_with_special_characters():
    """Passwords can contain special characters."""
    pass

def test_authenticate_user_password_is_case_sensitive():
    """Password matching is case-sensitive."""
    pass

# Username requirements
def test_authenticate_user_rejects_empty_username():
    """Empty username is not allowed."""
    pass

def test_authenticate_user_username_is_case_sensitive():
    """Username matching is case-sensitive."""
    pass

# Security
def test_authenticate_user_prevents_timing_attacks():
    """Authentication timing is constant regardless of username validity."""
    pass

def test_authenticate_user_does_not_leak_user_existence():
    """Error messages don't reveal whether username exists."""
    pass
```

Run pytest with verbose output:

```bash
pytest test_auth.py -v
```

**Output**:

```text
test_auth.py::test_authenticate_user_with_valid_credentials PASSED
test_auth.py::test_authenticate_user_rejects_invalid_password PASSED
test_auth.py::test_authenticate_user_rejects_nonexistent_user PASSED
test_auth.py::test_authenticate_user_rejects_empty_password PASSED
test_auth.py::test_authenticate_user_accepts_password_with_special_characters PASSED
test_auth.py::test_authenticate_user_password_is_case_sensitive PASSED
test_auth.py::test_authenticate_user_rejects_empty_username PASSED
test_auth.py::test_authenticate_user_username_is_case_sensitive PASSED
test_auth.py::test_authenticate_user_prevents_timing_attacks PASSED
test_auth.py::test_authenticate_user_does_not_leak_user_existence PASSED

========================== 10 passed in 0.03s ==========================
```

**Key insight**: Reading the test names is like reading a specification. A new developer can understand the authentication requirements without reading any code.

### Common Naming Anti-Patterns to Avoid

**Anti-pattern 1: Numbered tests**

```python
# Bad
def test_auth1():
    pass

def test_auth2():
    pass

# Good
def test_authenticate_user_with_valid_credentials():
    pass

def test_authenticate_user_rejects_invalid_password():
    pass
```

**Anti-pattern 2: Vague names**

```python
# Bad
def test_works():
    pass

def test_fails():
    pass

# Good
def test_authenticate_user_succeeds_with_valid_credentials():
    pass

def test_authenticate_user_fails_with_invalid_password():
    pass
```

**Anti-pattern 3: Implementation details in names**

```python
# Bad - tied to implementation
def test_authenticate_user_queries_database_and_compares_hashes():
    pass

# Good - describes behavior
def test_authenticate_user_verifies_password_against_stored_credentials():
    pass
```

**Anti-pattern 4: Testing multiple things**

```python
# Bad - tests multiple scenarios
def test_authentication_and_permissions_and_sessions():
    pass

# Good - separate tests
def test_authenticate_user_with_valid_credentials():
    pass

def test_check_user_permissions_after_authentication():
    pass

def test_create_session_after_successful_authentication():
    pass
```

### Naming Conventions Summary

**General pattern**:
```
test_<function_name>_<scenario>_<optional_outcome>
```

**Action verbs to use**:
- `accepts`, `rejects` - For validation
- `returns`, `raises` - For outcomes
- `handles`, `processes` - For operations
- `prevents`, `allows` - For security/permissions
- `creates`, `updates`, `deletes` - For CRUD operations

**Scenario descriptors**:
- `with_valid_<thing>` - Valid input
- `with_invalid_<thing>` - Invalid input
- `with_empty_<thing>` - Empty/null input
- `with_missing_<thing>` - Absent input
- `when_<condition>` - Conditional scenarios

**Outcome descriptors** (when needed):
- `returns_<value>` - Specific return value
- `raises_<error>` - Exception raised
- `succeeds` - Successful operation
- `fails` - Failed operation

### The Journey: From Generic to Descriptive

| Iteration | Naming Approach           | Problem                                | Solution                                    |
| --------- | ------------------------- | -------------------------------------- | ------------------------------------------- |
| 0         | Generic names             | No context, hard to debug              | Use descriptive names                       |
| 1         | Descriptive names         | Clear but potentially verbose          | Establish naming patterns                   |
| 2         | Naming patterns           | Need to cover different scenarios      | Pattern: function_scenario_outcome          |
| 3         | Balancing length          | Names getting too long                 | Include function and scenario, outcome optional |
| 4         | Different test types      | One pattern doesn't fit all            | Adapt pattern to test type                  |
| 5         | Documentation value       | Tests as specification                 | Names document behavior                     |

### Lessons Learned

**1. Names are documentation**: A well-named test tells you what it tests without reading the code

**2. Be specific about scenarios**: "invalid password" is better than "invalid input"

**3. Use action verbs**: "rejects", "accepts", "returns" make behavior clear

**4. Balance length and clarity**: Include function and scenario, outcome only when needed

**5. Adapt to test type**: Unit tests, integration tests, and edge cases benefit from different patterns

**6. Avoid implementation details**: Name tests by behavior, not by how they work internally

**7. Test names are specifications**: Reading test names should reveal the system's requirements

**8. Consistency matters**: Use the same patterns across your test suite for predictability

## Organizing Tests in Your Project

## Organizing Tests in Your Project

As your test suite grows from a handful of tests to hundreds or thousands, organization becomes critical. A well-organized test suite is easy to navigate, maintain, and run selectively. A poorly organized one becomes a maintenance burden that slows development.

Let's explore test organization through our authentication system as it scales from a simple module to a complex application.

### The Reference Problem: Scaling from Simple to Complex

Our authentication system has evolved into a full application with:
- User authentication (login, logout, password reset)
- Authorization (permissions, roles, access control)
- Session management (creation, validation, expiration)
- User management (registration, profile updates, deletion)

We need to organize hundreds of tests so they're easy to find, run, and maintain.

### Iteration 0: Everything in One File

Let's start with the naive approach—all tests in a single file.

```bash
# Project structure
my_project/
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── auth.py
│       ├── permissions.py
│       ├── sessions.py
│       └── users.py
└── tests/
    └── test_everything.py
```

```python
# tests/test_everything.py
from myapp.auth import authenticate_user, logout_user, reset_password
from myapp.permissions import check_permission, grant_permission
from myapp.sessions import create_session, validate_session
from myapp.users import register_user, update_profile, delete_user

# Authentication tests
def test_authenticate_user_with_valid_credentials():
    pass

def test_authenticate_user_rejects_invalid_password():
    pass

def test_logout_user_invalidates_session():
    pass

def test_reset_password_sends_email():
    pass

# Permission tests
def test_check_permission_for_admin():
    pass

def test_check_permission_for_regular_user():
    pass

def test_grant_permission_to_user():
    pass

# Session tests
def test_create_session_for_authenticated_user():
    pass

def test_validate_session_with_valid_token():
    pass

def test_validate_session_rejects_expired_token():
    pass

# User management tests
def test_register_user_with_valid_data():
    pass

def test_register_user_rejects_duplicate_email():
    pass

def test_update_profile_changes_email():
    pass

def test_delete_user_removes_all_data():
    pass

# ... 50 more tests ...
```

Run the tests:

```bash
pytest tests/test_everything.py -v
```

**Output** (truncated):

```text
tests/test_everything.py::test_authenticate_user_with_valid_credentials PASSED
tests/test_everything.py::test_authenticate_user_rejects_invalid_password PASSED
tests/test_everything.py::test_logout_user_invalidates_session PASSED
...
tests/test_everything.py::test_delete_user_removes_all_data PASSED

========================== 14 passed in 0.05s ==========================
```

**Current limitation**: This file is becoming unwieldy. Problems:
1. Hard to find specific tests (must scroll through hundreds of lines)
2. Can't run just authentication tests without running everything
3. Merge conflicts when multiple developers edit the same file
4. No clear organization—tests are just listed sequentially

**What we need**: A way to group related tests into separate files.

### Iteration 1: Splitting by Module

Let's organize tests to mirror our source code structure.

```bash
# Organized project structure
my_project/
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── auth.py
│       ├── permissions.py
│       ├── sessions.py
│       └── users.py
└── tests/
    ├── __init__.py
    ├── test_auth.py
    ├── test_permissions.py
    ├── test_sessions.py
    └── test_users.py
```

```python
# tests/test_auth.py
from myapp.auth import authenticate_user, logout_user, reset_password

def test_authenticate_user_with_valid_credentials():
    pass

def test_authenticate_user_rejects_invalid_password():
    pass

def test_logout_user_invalidates_session():
    pass

def test_reset_password_sends_email():
    pass
```

```python
# tests/test_permissions.py
from myapp.permissions import check_permission, grant_permission

def test_check_permission_for_admin():
    pass

def test_check_permission_for_regular_user():
    pass

def test_grant_permission_to_user():
    pass
```

```python
# tests/test_sessions.py
from myapp.sessions import create_session, validate_session

def test_create_session_for_authenticated_user():
    pass

def test_validate_session_with_valid_token():
    pass

def test_validate_session_rejects_expired_token():
    pass
```

```python
# tests/test_users.py
from myapp.users import register_user, update_profile, delete_user

def test_register_user_with_valid_data():
    pass

def test_register_user_rejects_duplicate_email():
    pass

def test_update_profile_changes_email():
    pass

def test_delete_user_removes_all_data():
    pass
```

Run all tests:

```bash
pytest tests/ -v
```

**Output**:

```text
tests/test_auth.py::test_authenticate_user_with_valid_credentials PASSED
tests/test_auth.py::test_authenticate_user_rejects_invalid_password PASSED
tests/test_auth.py::test_logout_user_invalidates_session PASSED
tests/test_auth.py::test_reset_password_sends_email PASSED
tests/test_permissions.py::test_check_permission_for_admin PASSED
tests/test_permissions.py::test_check_permission_for_regular_user PASSED
tests/test_permissions.py::test_grant_permission_to_user PASSED
tests/test_sessions.py::test_create_session_for_authenticated_user PASSED
tests/test_sessions.py::test_validate_session_with_valid_token PASSED
tests/test_sessions.py::test_validate_session_rejects_expired_token PASSED
tests/test_users.py::test_register_user_with_valid_data PASSED
tests/test_users.py::test_register_user_rejects_duplicate_email PASSED
tests/test_users.py::test_update_profile_changes_email PASSED
tests/test_users.py::test_delete_user_removes_all_data PASSED

========================== 14 passed in 0.05s ==========================
```

**Expected vs. Actual improvement**: Tests are now organized by module. We can:
- Find authentication tests in `test_auth.py`
- Run only authentication tests: `pytest tests/test_auth.py`
- See which module a test belongs to from the output

Now let's run just the authentication tests:

```bash
pytest tests/test_auth.py -v
```

**Output**:

```text
tests/test_auth.py::test_authenticate_user_with_valid_credentials PASSED
tests/test_auth.py::test_authenticate_user_rejects_invalid_password PASSED
tests/test_auth.py::test_logout_user_invalidates_session PASSED
tests/test_auth.py::test_reset_password_sends_email PASSED

========================== 4 passed in 0.02s ==========================
```

**Key insight**: Organizing tests by module makes selective test execution trivial. This is crucial for fast development cycles—you can run just the tests relevant to your current work.

**Current limitation**: Each test file is still growing large. What if `test_auth.py` has 50 tests covering different aspects of authentication?

### Iteration 2: Organizing by Feature Within Modules

Let's further organize tests by creating subdirectories for complex modules.

```bash
# Hierarchical project structure
my_project/
├── src/
│   └── myapp/
│       ├── __init__.py
│       ├── auth/
│       │   ├── __init__.py
│       │   ├── authentication.py
│       │   ├── password_reset.py
│       │   └── logout.py
│       ├── permissions.py
│       ├── sessions.py
│       └── users.py
└── tests/
    ├── __init__.py
    ├── auth/
    │   ├── __init__.py
    │   ├── test_authentication.py
    │   ├── test_password_reset.py
    │   └── test_logout.py
    ├── test_permissions.py
    ├── test_sessions.py
    └── test_users.py
```

```python
# tests/auth/test_authentication.py
from myapp.auth.authentication import authenticate_user, verify_credentials

def test_authenticate_user_with_valid_credentials():
    pass

def test_authenticate_user_rejects_invalid_password():
    pass

def test_authenticate_user_rejects_nonexistent_user():
    pass

def test_verify_credentials_with_hashed_password():
    pass
```

```python
# tests/auth/test_password_reset.py
from myapp.auth.password_reset import request_reset, validate_reset_token, reset_password

def test_request_reset_sends_email():
    pass

def test_request_reset_generates_token():
    pass

def test_validate_reset_token_accepts_valid_token():
    pass

def test_validate_reset_token_rejects_expired_token():
    pass

def test_reset_password_updates_credentials():
    pass
```

```python
# tests/auth/test_logout.py
from myapp.auth.logout import logout_user, invalidate_all_sessions

def test_logout_user_invalidates_current_session():
    pass

def test_logout_user_preserves_other_sessions():
    pass

def test_invalidate_all_sessions_removes_all_user_sessions():
    pass
```

Run all tests:

```bash
pytest tests/ -v
```

**Output**:

```text
tests/auth/test_authentication.py::test_authenticate_user_with_valid_credentials PASSED
tests/auth/test_authentication.py::test_authenticate_user_rejects_invalid_password PASSED
tests/auth/test_authentication.py::test_authenticate_user_rejects_nonexistent_user PASSED
tests/auth/test_authentication.py::test_verify_credentials_with_hashed_password PASSED
tests/auth/test_password_reset.py::test_request_reset_sends_email PASSED
tests/auth/test_password_reset.py::test_request_reset_generates_token PASSED
tests/auth/test_password_reset.py::test_validate_reset_token_accepts_valid_token PASSED
tests/auth/test_password_reset.py::test_validate_reset_token_rejects_expired_token PASSED
tests/auth/test_password_reset.py::test_reset_password_updates_credentials PASSED
tests/auth/test_logout.py::test_logout_user_invalidates_current_session PASSED
tests/auth/test_logout.py::test_logout_user_preserves_other_sessions PASSED
tests/auth/test_logout.py::test_invalidate_all_sessions_removes_all_user_sessions PASSED
tests/test_permissions.py::test_check_permission_for_admin PASSED
tests/test_permissions.py::test_check_permission_for_regular_user PASSED
tests/test_permissions.py::test_grant_permission_to_user PASSED
tests/test_sessions.py::test_create_session_for_authenticated_user PASSED
tests/test_sessions.py::test_validate_session_with_valid_token PASSED
tests/test_sessions.py::test_validate_session_rejects_expired_token PASSED
tests/test_users.py::test_register_user_with_valid_data PASSED
tests/test_users.py::test_register_user_rejects_duplicate_email PASSED
tests/test_users.py::test_update_profile_changes_email PASSED
tests/test_users.py::test_delete_user_removes_all_data PASSED

========================== 22 passed in 0.08s ==========================
```

**Expected vs. Actual improvement**: Now we can run tests at different levels of granularity:

```bash
# Run all tests
pytest tests/

# Run all auth tests
pytest tests/auth/

# Run only authentication tests
pytest tests/auth/test_authentication.py

# Run only password reset tests
pytest tests/auth/test_password_reset.py
```

**Key insight**: Hierarchical organization mirrors your source code structure and enables precise test selection. This is essential for large projects where running the entire suite takes minutes or hours.

### Iteration 3: Organizing by Test Type

Sometimes you want to organize tests by type rather than by module. Let's add integration tests and performance tests.

```bash
# Project structure with test types
my_project/
├── src/
│   └── myapp/
│       └── ...
└── tests/
    ├── __init__.py
    ├── unit/
    │   ├── __init__.py
    │   ├── auth/
    │   │   ├── test_authentication.py
    │   │   └── test_password_reset.py
    │   ├── test_permissions.py
    │   └── test_sessions.py
    ├── integration/
    │   ├── __init__.py
    │   ├── test_login_workflow.py
    │   ├── test_registration_workflow.py
    │   └── test_password_reset_workflow.py
    └── performance/
        ├── __init__.py
        ├── test_authentication_performance.py
        └── test_session_performance.py
```

```python
# tests/unit/auth/test_authentication.py
from myapp.auth.authentication import authenticate_user

def test_authenticate_user_with_valid_credentials():
    """Unit test: tests authentication function in isolation."""
    pass
```

```python
# tests/integration/test_login_workflow.py
from myapp.auth.authentication import authenticate_user
from myapp.sessions import create_session
from myapp.permissions import load_user_permissions

def test_complete_login_workflow():
    """Integration test: tests authentication + session + permissions."""
    # Authenticate
    user = authenticate_user("alice", "secret123", users_db)
    assert user is not None
    
    # Create session
    session = create_session(user)
    assert session["user_id"] == user["id"]
    
    # Load permissions
    permissions = load_user_permissions(user)
    assert "read" in permissions
```

```python
# tests/performance/test_authentication_performance.py
import time
from myapp.auth.authentication import authenticate_user

def test_authentication_completes_within_100ms():
    """Performance test: verifies authentication speed."""
    users = create_large_user_database(10000)
    
    start = time.time()
    authenticate_user("user5000", "password", users)
    duration = time.time() - start
    
    assert duration < 0.1, f"Authentication took {duration}s, expected < 0.1s"
```

Now we can run tests by type:

```bash
# Run only unit tests (fast)
pytest tests/unit/ -v

# Run only integration tests (slower)
pytest tests/integration/ -v

# Run only performance tests (slowest)
pytest tests/performance/ -v

# Run unit and integration tests, skip performance
pytest tests/unit/ tests/integration/ -v
```

**Expected vs. Actual improvement**: Organizing by test type enables:
- Running fast unit tests during development
- Running integration tests before committing
- Running performance tests only in CI or before releases

**When to use this organization**:
- When you have tests with significantly different execution times
- When you want to run different test types in different CI stages
- When you have tests that require different environments (e.g., performance tests need production-like hardware)

### Iteration 4: Sharing Test Utilities with conftest.py

As tests grow, you'll need to share fixtures and utilities. Pytest provides `conftest.py` for this purpose.

```bash
# Project structure with conftest.py
my_project/
├── src/
│   └── myapp/
│       └── ...
└── tests/
    ├── __init__.py
    ├── conftest.py              # Shared across all tests
    ├── auth/
    │   ├── __init__.py
    │   ├── conftest.py          # Shared across auth tests
    │   ├── test_authentication.py
    │   └── test_password_reset.py
    └── test_permissions.py
```

```python
# tests/conftest.py
import pytest

@pytest.fixture
def sample_users():
    """Fixture available to all tests."""
    return {
        "alice": ("secret123", {"id": 1, "email": "alice@example.com"}),
        "bob": ("password456", {"id": 2, "email": "bob@example.com"})
    }

@pytest.fixture
def admin_user():
    """Fixture for admin user, available to all tests."""
    return ("admin123", {
        "id": 999,
        "email": "admin@example.com",
        "role": "admin",
        "permissions": ["read", "write", "delete", "admin"]
    })
```

```python
# tests/auth/conftest.py
import pytest

@pytest.fixture
def authenticated_session(sample_users):
    """Fixture available only to auth tests."""
    from myapp.auth.authentication import authenticate_user
    from myapp.sessions import create_session
    
    user = authenticate_user("alice", "secret123", sample_users)
    return create_session(user)
```

```python
# tests/auth/test_authentication.py
def test_authenticate_user_with_valid_credentials(sample_users):
    """Uses fixture from tests/conftest.py."""
    from myapp.auth.authentication import authenticate_user
    result = authenticate_user("alice", "secret123", sample_users)
    assert result is not None

def test_logout_invalidates_session(authenticated_session):
    """Uses fixture from tests/auth/conftest.py."""
    from myapp.auth.logout import logout_user
    logout_user(authenticated_session)
    assert authenticated_session["valid"] == False
```

**Expected vs. Actual improvement**: `conftest.py` files provide:
- Shared fixtures without explicit imports
- Hierarchical fixture scope (directory-level fixtures)
- Centralized test utilities

**Key insight**: Fixtures in `conftest.py` are automatically discovered and available to tests in the same directory and subdirectories. We'll explore fixtures in depth in Chapter 4.

### Iteration 5: Organizing for Large Projects

For very large projects, you might need additional organization strategies.

```bash
# Large project structure
my_project/
├── src/
│   └── myapp/
│       ├── api/
│       ├── auth/
│       ├── database/
│       ├── services/
│       └── utils/
└── tests/
    ├── conftest.py
    ├── fixtures/                # Shared test data
    │   ├── __init__.py
    │   ├── users.py
    │   ├── sessions.py
    │   └── permissions.py
    ├── helpers/                 # Test utilities
    │   ├── __init__.py
    │   ├── assertions.py
    │   ├── builders.py
    │   └── mocks.py
    ├── unit/
    │   ├── conftest.py
    │   ├── api/
    │   ├── auth/
    │   ├── database/
    │   ├── services/
    │   └── utils/
    ├── integration/
    │   ├── conftest.py
    │   ├── test_api_workflows.py
    │   ├── test_auth_workflows.py
    │   └── test_database_workflows.py
    └── e2e/                     # End-to-end tests
        ├── conftest.py
        ├── test_user_registration.py
        ├── test_user_login.py
        └── test_user_profile.py
```

```python
# tests/fixtures/users.py
"""Shared user fixtures for all tests."""

def create_test_user(username="alice", **kwargs):
    """Builder function for test users."""
    return {
        "username": username,
        "email": kwargs.get("email", f"{username}@example.com"),
        "id": kwargs.get("id", 1),
        "role": kwargs.get("role", "user")
    }

def create_admin_user(**kwargs):
    """Builder function for admin users."""
    return create_test_user(role="admin", **kwargs)
```

```python
# tests/helpers/assertions.py
"""Custom assertion helpers."""

def assert_valid_user_profile(profile):
    """Verify a user profile has all required fields."""
    assert "id" in profile
    assert "email" in profile
    assert "username" in profile
    assert "@" in profile["email"]

def assert_authentication_failed(result):
    """Verify authentication failed as expected."""
    assert result is None, "Expected authentication to fail"
```

```python
# tests/unit/auth/test_authentication.py
from tests.fixtures.users import create_test_user
from tests.helpers.assertions import assert_valid_user_profile

def test_authenticate_user_with_valid_credentials():
    user = create_test_user("alice")
    # ... test code ...
    assert_valid_user_profile(result)
```

**Expected vs. Actual improvement**: This structure provides:
- Clear separation of test types (unit, integration, e2e)
- Centralized test utilities (fixtures, helpers)
- Scalability for hundreds or thousands of tests

### Organization Strategies Summary

**By module** (mirrors source code):

```bash
tests/
├── test_auth.py
├── test_permissions.py
└── test_sessions.py
```

**By feature** (hierarchical):

```bash
tests/
├── auth/
│   ├── test_authentication.py
│   ├── test_password_reset.py
│   └── test_logout.py
├── test_permissions.py
└── test_sessions.py
```

**By test type**:

```bash
tests/
├── unit/
├── integration/
└── performance/
```

**Hybrid approach** (recommended for large projects):

```bash
tests/
├── fixtures/          # Shared test data
├── helpers/           # Test utilities
├── unit/
│   ├── auth/
│   ├── permissions/
│   └── sessions/
├── integration/
└── e2e/
```

### Best Practices for Test Organization

**1. Mirror your source structure**: Tests should be easy to find by matching source code organization

**2. Use conftest.py for shared fixtures**: Don't duplicate fixture code across test files

**3. Separate by test type when execution time varies**: Fast unit tests separate from slow integration tests

**4. Keep test files focused**: Each test file should test one module or feature

**5. Use subdirectories for complex modules**: Don't let test files grow beyond ~500 lines

**6. Create helper modules for common utilities**: Centralize assertion helpers, builders, and mocks

**7. Document your organization**: Add a README in your tests directory explaining the structure

### The Journey: From Chaos to Structure

| Iteration | Organization Approach         | Problem                                | Solution                                    |
| --------- | ----------------------------- | -------------------------------------- | ------------------------------------------- |
| 0         | Single file                   | Hard to navigate, can't run selectively | Split by module                             |
| 1         | Split by module               | Large modules still unwieldy           | Create subdirectories for features          |
| 2         | Hierarchical by feature       | Can't separate fast/slow tests         | Organize by test type                       |
| 3         | Organize by test type         | Duplicated fixtures and utilities      | Use conftest.py for sharing                 |
| 4         | Shared utilities              | Large projects need more structure     | Hybrid approach with fixtures/helpers dirs  |
| 5         | Large project structure       | Scalable organization                  | Clear separation of concerns                |

### Lessons Learned

**1. Start simple, evolve as needed**: Begin with one test file per module, add structure as complexity grows

**2. Organization enables selective execution**: Good structure lets you run just the tests you need

**3. Mirror source code structure**: Tests are easier to find when they match source organization

**4. Separate by execution time**: Fast tests in one place, slow tests in another

**5. Share fixtures via conftest.py**: Avoid duplication, maintain consistency

**6. Create helper modules**: Centralize common test utilities

**7. Document your structure**: Help new developers understand your organization

**8. Consistency matters**: Use the same patterns across your entire test suite

## Running Specific Tests

## Running Specific Tests

Running your entire test suite every time you make a change is slow and inefficient. Pytest provides powerful mechanisms for running exactly the tests you need—whether that's a single test, a group of related tests, or tests matching specific criteria.

Let's explore test selection through our authentication system, discovering how to run tests efficiently during development.

### The Reference Problem: Selective Test Execution

Our authentication system now has:
- 50+ unit tests across multiple modules
- 20+ integration tests
- 10+ performance tests
- Tests that take milliseconds and tests that take seconds

We need to run tests selectively to maintain fast development cycles.

### Iteration 0: Running Everything

Let's start with the default—running all tests.

```bash
# Project structure
my_project/
└── tests/
    ├── auth/
    │   ├── test_authentication.py  (10 tests)
    │   ├── test_password_reset.py  (8 tests)
    │   └── test_logout.py          (5 tests)
    ├── test_permissions.py         (12 tests)
    ├── test_sessions.py            (15 tests)
    └── test_users.py               (10 tests)
```

Run all tests:

```bash
cd my_project
pytest
```

**Output**:

```text
========================== test session starts ==========================
collected 60 items

tests/auth/test_authentication.py ..........                           [ 16%]
tests/auth/test_password_reset.py ........                             [ 30%]
tests/auth/test_logout.py .....                                        [ 38%]
tests/test_permissions.py ............                                 [ 58%]
tests/test_sessions.py ...............                                 [ 83%]
tests/test_users.py ..........                                         [100%]

========================== 60 passed in 2.45s ===========================
```

**Current limitation**: Running all 60 tests takes 2.45 seconds. During development, we're only working on authentication—we don't need to run permission, session, and user tests. We need faster feedback.

**What we need**: Ways to run subsets of tests.

### Iteration 1: Running Tests by File

The simplest way to run specific tests is to specify the file.

```bash
# Run only authentication tests
pytest tests/auth/test_authentication.py
```

**Output**:

```text
========================== test session starts ==========================
collected 10 items

tests/auth/test_authentication.py ..........                           [100%]

========================== 10 passed in 0.42s ===========================
```

**Expected vs. Actual improvement**: Running just the authentication tests takes 0.42s instead of 2.45s—nearly 6x faster. This is the most common way to run specific tests during development.

You can also run multiple files:

```bash
# Run authentication and password reset tests
pytest tests/auth/test_authentication.py tests/auth/test_password_reset.py
```

**Output**:

```text
========================== test session starts ==========================
collected 18 items

tests/auth/test_authentication.py ..........                           [ 55%]
tests/auth/test_password_reset.py ........                             [100%]

========================== 18 passed in 0.78s ===========================
```

**Current limitation**: What if we want to run just one specific test function, not the entire file?

### Iteration 2: Running a Specific Test Function

Pytest allows you to specify individual test functions using the `::` syntax.

```python
# tests/auth/test_authentication.py
def test_authenticate_user_with_valid_credentials():
    pass

def test_authenticate_user_rejects_invalid_password():
    pass

def test_authenticate_user_rejects_nonexistent_user():
    pass
```

Run just one test:

```bash
pytest tests/auth/test_authentication.py::test_authenticate_user_with_valid_credentials
```

**Output**:

```text
========================== test session starts ==========================
collected 1 item

tests/auth/test_authentication.py::test_authenticate_user_with_valid_credentials PASSED [100%]

========================== 1 passed in 0.08s ============================
```

**Expected vs. Actual improvement**: Running a single test takes only 0.08s. This is perfect for rapid iteration when fixing a specific bug.

**Syntax**: `<file_path>::<test_function_name>`

You can also run multiple specific tests:

```bash
pytest tests/auth/test_authentication.py::test_authenticate_user_with_valid_credentials \
       tests/auth/test_authentication.py::test_authenticate_user_rejects_invalid_password
```

**Current limitation**: Typing out full test paths is tedious. Is there a way to select tests by pattern?

### Iteration 3: Running Tests by Name Pattern

Pytest's `-k` flag lets you run tests matching a name pattern.

```python
# tests/auth/test_authentication.py
def test_authenticate_user_with_valid_credentials():
    pass

def test_authenticate_user_rejects_invalid_password():
    pass

def test_authenticate_user_rejects_nonexistent_user():
    pass

def test_verify_password_hash():
    pass
```

Run all tests with "invalid" in the name:

```bash
pytest -k "invalid"
```

**Output**:

```text
========================== test session starts ==========================
collected 60 items / 58 deselected / 2 selected

tests/auth/test_authentication.py::test_authenticate_user_rejects_invalid_password PASSED [ 50%]
tests/test_permissions.py::test_check_permission_rejects_invalid_permission PASSED [100%]

==================== 2 passed, 58 deselected in 0.15s ===================
```

**Expected vs. Actual improvement**: The `-k` flag found all tests with "invalid" in their name across all files. Notice:
- `collected 60 items` - Pytest found all tests
- `58 deselected / 2 selected` - It filtered to just the matching ones
- Only 2 tests ran

**Pattern matching examples**:

```bash
# Run tests with "password" in the name
pytest -k "password"

# Run tests with "authenticate" OR "verify" in the name
pytest -k "authenticate or verify"

# Run tests with "user" AND "valid" in the name
pytest -k "user and valid"

# Run tests with "user" but NOT "invalid" in the name
pytest -k "user and not invalid"
```

Let's see a more complex example:

```bash
pytest -k "authenticate and (valid or invalid)"
```

**Output**:

```text
========================== test session starts ==========================
collected 60 items / 57 deselected / 3 selected

tests/auth/test_authentication.py::test_authenticate_user_with_valid_credentials PASSED [ 33%]
tests/auth/test_authentication.py::test_authenticate_user_rejects_invalid_password PASSED [ 66%]
tests/auth/test_authentication.py::test_authenticate_user_with_invalid_token PASSED [100%]

==================== 3 passed, 57 deselected in 0.18s ===================
```

**Key insight**: The `-k` flag is incredibly powerful for running related tests without specifying exact paths. It's perfect for:
- Running all tests related to a feature ("password", "session", "permission")
- Running all tests for a specific scenario ("valid", "invalid", "expired")
- Excluding certain tests ("not slow", "not integration")

**Current limitation**: What if we want to run tests in a specific directory without typing the full path?

### Iteration 4: Running Tests by Directory

You can run all tests in a directory by specifying the directory path.

```bash
# Run all auth tests
pytest tests/auth/
```

**Output**:

```text
========================== test session starts ==========================
collected 23 items

tests/auth/test_authentication.py ..........                           [ 43%]
tests/auth/test_password_reset.py ........                             [ 78%]
tests/auth/test_logout.py .....                                        [100%]

========================== 23 passed in 0.95s ===========================
```

**Expected vs. Actual improvement**: Running all tests in the `auth/` directory is simple and fast. This is useful when working on a specific module.

You can combine directory paths with other selection methods:

```bash
# Run tests in auth/ directory that match "invalid"
pytest tests/auth/ -k "invalid"
```

**Output**:

```text
========================== test session starts ==========================
collected 23 items / 22 deselected / 1 selected

tests/auth/test_authentication.py::test_authenticate_user_rejects_invalid_password PASSED [100%]

==================== 1 passed, 22 deselected in 0.12s ===================
```

**Current limitation**: What if we want to run tests that failed in the last run?

### Iteration 5: Running Failed Tests

Pytest can remember which tests failed and rerun only those.

```python
# tests/auth/test_authentication.py
def test_authenticate_user_with_valid_credentials():
    assert True  # Passes

def test_authenticate_user_rejects_invalid_password():
    assert False  # Fails!

def test_authenticate_user_rejects_nonexistent_user():
    assert True  # Passes
```

First, run all tests to see which fail:

```bash
pytest tests/auth/test_authentication.py
```

**Output**:

```text
========================== test session starts ==========================
collected 3 items

tests/auth/test_authentication.py .F.                                  [100%]

=========================== FAILURES ====================================
________ test_authenticate_user_rejects_invalid_password ________________

    def test_authenticate_user_rejects_invalid_password():
>       assert False
E       assert False

tests/auth/test_authentication.py:5: AssertionError
==================== 1 failed, 2 passed in 0.12s ========================
```

Now run only the failed test:

```bash
pytest --lf  # --lf means "last failed"
```

**Output**:

```text
========================== test session starts ==========================
collected 3 items / 2 deselected / 1 selected
run-last-failure: rerun previous 1 failure

tests/auth/test_authentication.py F                                    [100%]

=========================== FAILURES ====================================
________ test_authenticate_user_rejects_invalid_password ________________

    def test_authenticate_user_rejects_invalid_password():
>       assert False
E       assert False

tests/auth/test_authentication.py:5: AssertionError
==================== 1 failed, 2 deselected in 0.08s ====================
```

**Expected vs. Actual improvement**: Pytest remembered which test failed and ran only that one. Notice:
- `collected 3 items / 2 deselected / 1 selected` - Found all tests but ran only the failed one
- `run-last-failure: rerun previous 1 failure` - Explicit confirmation

**Related flags**:

```bash
# Run last failed tests first, then the rest
pytest --ff  # --ff means "failed first"

# Run only tests that failed, then stop
pytest --lf --exitfirst  # Stop after first failure
```

Let's see `--ff` in action:

```bash
pytest --ff tests/auth/test_authentication.py
```

**Output**:

```text
========================== test session starts ==========================
collected 3 items
run-last-failure: rerun previous 1 failure first

tests/auth/test_authentication.py F..                                  [100%]

=========================== FAILURES ====================================
________ test_authenticate_user_rejects_invalid_password ________________

    def test_authenticate_user_rejects_invalid_password():
>       assert False
E       assert False

tests/auth/test_authentication.py:5: AssertionError
==================== 1 failed, 2 passed in 0.12s ========================
```

**Key insight**: The failed test ran first (F), then the passing tests (..). This is useful when you want to verify your fix works before running the full suite.

### Iteration 6: Stopping on First Failure

When debugging, you often want to stop as soon as a test fails.

```python
# tests/auth/test_authentication.py
def test_1():
    assert False  # Fails

def test_2():
    assert False  # Would fail, but won't run

def test_3():
    assert True   # Would pass, but won't run
```

Run with `--exitfirst` (or `-x`):

```bash
pytest tests/auth/test_authentication.py -x
```

**Output**:

```text
========================== test session starts ==========================
collected 3 items

tests/auth/test_authentication.py F

=========================== FAILURES ====================================
__________________________ test_1 _______________________________________

    def test_1():
>       assert False
E       assert False

tests/auth/test_authentication.py:2: AssertionError
==================== 1 failed in 0.08s ==================================
```

**Expected vs. Actual improvement**: Pytest stopped after the first failure. Only `test_1` ran; `test_2` and `test_3` were never executed.

**Why this matters**: When debugging, you want immediate feedback on the first failure. Running subsequent tests wastes time and clutters the output.

You can also limit the number of failures:

```bash
# Stop after 3 failures
pytest --maxfail=3
```

### Iteration 7: Running Tests Modified Since Last Commit

Pytest can run only tests in files that have changed since the last git commit.

```bash
# Modify a test file
echo "def test_new(): pass" >> tests/auth/test_authentication.py

# Run only tests in modified files
pytest --testmon
```

**Note**: This requires the `pytest-testmon` plugin:

```bash
pip install pytest-testmon
```

**Alternative**: Use git to find modified files and run their tests:

```bash
# Run tests in files modified since last commit
pytest $(git diff --name-only HEAD | grep test_)
```

### Common Test Selection Patterns

**By file**:

```bash
pytest tests/test_auth.py
pytest tests/auth/test_authentication.py tests/auth/test_logout.py
```

**By directory**:

```bash
pytest tests/auth/
pytest tests/unit/ tests/integration/
```

**By function**:

```bash
pytest tests/test_auth.py::test_authenticate_user_with_valid_credentials
```

**By name pattern**:

```bash
pytest -k "password"
pytest -k "authenticate and valid"
pytest -k "not slow"
```

**By failure status**:

```bash
pytest --lf              # Last failed
pytest --ff              # Failed first
pytest --lf -x           # Last failed, stop on first failure
```

**Combining selections**:

```bash
pytest tests/auth/ -k "invalid" --lf
# Run tests in auth/ directory, matching "invalid", that failed last time
```

### Useful Flags for Test Selection

**Verbosity**:

```bash
pytest -v                # Verbose: show each test name
pytest -vv               # Extra verbose: show more details
pytest -q                # Quiet: minimal output
```

**Collection**:

```bash
pytest --collect-only    # Show what would run, don't run tests
pytest --co              # Short form of --collect-only
```

**Execution control**:

```bash
pytest -x                # Stop on first failure
pytest --maxfail=3       # Stop after 3 failures
pytest --lf              # Run last failed tests
pytest --ff              # Run failed tests first
```

**Output control**:

```bash
pytest -s                # Show print statements (don't capture output)
pytest --tb=short        # Short traceback format
pytest --tb=no           # No traceback, just summary
```

### Development Workflow Examples

**Rapid iteration on a single test**:

```bash
# Edit test, run it, repeat
pytest tests/auth/test_authentication.py::test_authenticate_user_with_valid_credentials -v
```

**Fix a failing test**:

```bash
# Run all tests to see what fails
pytest

# Run only failed tests
pytest --lf -x

# Fix the code, rerun failed test
pytest --lf

# Once it passes, run all tests
pytest
```

**Work on a feature**:

```bash
# Run tests related to the feature
pytest -k "password_reset"

# Run tests in the feature's directory
pytest tests/auth/

# Run fast, then slow tests
pytest tests/unit/auth/
pytest tests/integration/
```

**Pre-commit check**:

```bash
# Run all tests, stop on first failure
pytest -x

# Or run all tests, show summary
pytest -v
```

### The Journey: From All to Specific

| Iteration | Selection Method              | Use Case                                | Speed Improvement |
| --------- | ----------------------------- | --------------------------------------- | ----------------- |
| 0         | Run all tests                 | Full validation                         | Baseline (2.45s)  |
| 1         | Run by file                   | Working on specific module              | 6x faster (0.42s) |
| 2         | Run specific function         | Fixing specific bug                     | 30x faster (0.08s)|
| 3         | Run by name pattern           | Testing related functionality           | Variable          |
| 4         | Run by directory              | Working on module group                 | 2-3x faster       |
| 5         | Run failed tests              | Iterating on fixes                      | Variable          |
| 6         | Stop on first failure         | Debugging                               | Immediate feedback|
| 7         | Run modified tests            | Pre-commit check                        | Variable          |

### Lessons Learned

**1. Selective execution is essential**: Running all tests every time is too slow for development

**2. Use file paths for precision**: Most specific and predictable selection method

**3. Use `-k` for flexibility**: Pattern matching is powerful for related tests

**4. Use `--lf` when debugging**: Focus on failed tests until they pass

**5. Use `-x` to stop early**: Get immediate feedback on first failure

**6. Combine selection methods**: Directory + pattern + failure status = precise control

**7. Learn the shortcuts**: `-v`, `-x`, `--lf`, `-k` are your daily tools

**8. Optimize your workflow**: Fast feedback loops make development more productive
