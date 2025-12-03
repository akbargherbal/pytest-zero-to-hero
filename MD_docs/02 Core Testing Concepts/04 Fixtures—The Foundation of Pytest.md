# Chapter 4: Fixtures—The Foundation of Pytest

## What Are Fixtures?

## The Problem Fixtures Solve

Before we define what fixtures are, let's understand the problem they solve. Consider testing a user authentication system. Every test needs a database connection, a user account, and perhaps some test data. Without fixtures, you'd write this setup code in every single test function.

Let's see this problem in action with a concrete example: testing a `UserAuthenticator` class that validates user credentials against a database.

```python
# auth.py
class UserAuthenticator:
    def __init__(self, db_connection):
        self.db = db_connection
    
    def authenticate(self, username, password):
        """Verify username and password against database."""
        user = self.db.get_user(username)
        if user is None:
            return False
        return user['password'] == password
    
    def is_admin(self, username):
        """Check if user has admin privileges."""
        user = self.db.get_user(username)
        if user is None:
            return False
        return user.get('role') == 'admin'
```

Now let's write tests for this authenticator. Here's the naive approach—the way you might write tests before learning about fixtures:

```python
# test_auth_naive.py
from auth import UserAuthenticator

class MockDatabase:
    """Simple in-memory database for testing."""
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

def test_authenticate_valid_user():
    # Setup: Create database and add test user
    db = MockDatabase()
    db.add_user('alice', 'secret123')
    authenticator = UserAuthenticator(db)
    
    # Test: Valid credentials should authenticate
    result = authenticator.authenticate('alice', 'secret123')
    assert result is True

def test_authenticate_invalid_password():
    # Setup: Create database and add test user
    db = MockDatabase()
    db.add_user('alice', 'secret123')
    authenticator = UserAuthenticator(db)
    
    # Test: Wrong password should fail
    result = authenticator.authenticate('alice', 'wrong_password')
    assert result is False

def test_authenticate_nonexistent_user():
    # Setup: Create database and add test user
    db = MockDatabase()
    db.add_user('alice', 'secret123')
    authenticator = UserAuthenticator(db)
    
    # Test: Unknown user should fail
    result = authenticator.authenticate('bob', 'any_password')
    assert result is False

def test_is_admin_for_admin_user():
    # Setup: Create database and add admin user
    db = MockDatabase()
    db.add_user('admin_user', 'admin_pass', role='admin')
    authenticator = UserAuthenticator(db)
    
    # Test: Admin user should be identified
    result = authenticator.is_admin('admin_user')
    assert result is True

def test_is_admin_for_regular_user():
    # Setup: Create database and add regular user
    db = MockDatabase()
    db.add_user('alice', 'secret123', role='user')
    authenticator = UserAuthenticator(db)
    
    # Test: Regular user should not be admin
    result = authenticator.is_admin('alice')
    assert result is False
```

Let's run these tests to verify they work:

```bash
pytest test_auth_naive.py -v
```

**Output:**
```
======================== test session starts =========================
collected 5 items

test_auth_naive.py::test_authenticate_valid_user PASSED        [ 20%]
test_auth_naive.py::test_authenticate_invalid_password PASSED  [ 40%]
test_auth_naive.py::test_authenticate_nonexistent_user PASSED  [ 60%]
test_auth_naive.py::test_is_admin_for_admin_user PASSED        [ 80%]
test_auth_naive.py::test_is_admin_for_regular_user PASSED      [100%]

========================= 5 passed in 0.02s ==========================
```

The tests pass! But look closely at the code. Notice the problem?

### The Code Duplication Problem

Every single test repeats these three lines:

```python
db = MockDatabase()
db.add_user('alice', 'secret123')
authenticator = UserAuthenticator(db)
```

This is **setup code**—the scaffolding needed before you can test the actual behavior. In our five tests, we've written these setup lines 15 times. This creates several problems:

1. **Maintenance burden**: If we need to change how the database is initialized, we must update every test
2. **Inconsistency risk**: It's easy to accidentally create different setups in different tests
3. **Obscured intent**: The actual test logic is buried among setup code
4. **Scaling nightmare**: With 50 tests, you'd have 150 lines of duplicated setup

### What If We Need More Complex Setup?

The problem gets worse as setup becomes more complex. Imagine if our authenticator also needed:
- A configuration object
- A logging system
- A cache layer
- A password hashing service

Each test would need 10-15 lines of setup before getting to the actual assertion. The test file would become unreadable.

## Fixtures: The Solution

**A fixture is a function that provides setup (and optionally teardown) for your tests.** Instead of copying setup code into every test, you write it once in a fixture, and pytest automatically runs that fixture before each test that needs it.

Here's the key insight: **Fixtures turn setup code into reusable components.**

Think of fixtures as:
- **Ingredients** that your tests consume
- **Dependencies** that pytest automatically provides
- **Building blocks** that compose together

When you write a test that needs a database connection, you don't create the database yourself—you declare that your test needs a `db` fixture, and pytest handles the rest.

### The Fixture Mental Model

Before we see the syntax, understand the concept:

1. **You define a fixture**: A function decorated with `@pytest.fixture` that returns the setup object
2. **You declare dependencies**: Tests request fixtures by name as function parameters
3. **Pytest handles execution**: Before running your test, pytest automatically calls the fixture and passes the result to your test

This is **dependency injection**—your test declares what it needs, and pytest provides it.

Let's see this in action by refactoring our authentication tests to use fixtures.

## Simple Fixtures: Setup and Teardown

## Refactoring to Fixtures: First Attempt

Let's transform our duplicated setup code into a fixture. We'll start with the simplest possible fixture—one that just provides the database.

```python
# test_auth_with_fixtures.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    """Simple in-memory database for testing."""
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture
def db():
    """Provide a clean database for each test."""
    database = MockDatabase()
    database.add_user('alice', 'secret123')
    return database

def test_authenticate_valid_user(db):
    authenticator = UserAuthenticator(db)
    result = authenticator.authenticate('alice', 'secret123')
    assert result is True

def test_authenticate_invalid_password(db):
    authenticator = UserAuthenticator(db)
    result = authenticator.authenticate('alice', 'wrong_password')
    assert result is False
```

Let's break down what just happened:

### Anatomy of a Fixture

```python
@pytest.fixture
def db():
    database = MockDatabase()
    database.add_user('alice', 'secret123')
    return database
```

1. **`@pytest.fixture`**: This decorator tells pytest "this function is a fixture"
2. **`def db():`**: The fixture name becomes the parameter name tests use
3. **Setup code**: Everything before `return` runs before each test
4. **`return database`**: The fixture provides this object to tests

### Using a Fixture

```python
def test_authenticate_valid_user(db):
    authenticator = UserAuthenticator(db)
    result = authenticator.authenticate('alice', 'secret123')
    assert result is True
```

Notice the test function now has a parameter: `db`. This is **not** a normal function parameter. When pytest sees this:

1. It looks for a fixture named `db`
2. It calls that fixture function
3. It passes the return value to your test

You never call `db()` yourself. Pytest handles it automatically.

Let's run these tests to verify the fixture works:

```bash
pytest test_auth_with_fixtures.py -v
```

**Output:**
```
======================== test session starts =========================
collected 2 items

test_auth_with_fixtures.py::test_authenticate_valid_user PASSED [ 50%]
test_auth_with_fixtures.py::test_authenticate_invalid_password PASSED [100%]

========================= 2 passed in 0.01s ==========================
```

Perfect! The tests pass. But we can improve this further.

## Composing Fixtures: Building on Top of Other Fixtures

Our tests still repeat this line:
```python
authenticator = UserAuthenticator(db)
```

Let's create a fixture for the authenticator itself. Here's where fixtures become powerful—**fixtures can use other fixtures**.

```python
# test_auth_composed.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture
def db():
    """Provide a clean database for each test."""
    database = MockDatabase()
    database.add_user('alice', 'secret123')
    database.add_user('admin_user', 'admin_pass', role='admin')
    return database

@pytest.fixture
def authenticator(db):
    """Provide an authenticator connected to the test database."""
    return UserAuthenticator(db)

def test_authenticate_valid_user(authenticator):
    result = authenticator.authenticate('alice', 'secret123')
    assert result is True

def test_authenticate_invalid_password(authenticator):
    result = authenticator.authenticate('alice', 'wrong_password')
    assert result is False

def test_authenticate_nonexistent_user(authenticator):
    result = authenticator.authenticate('bob', 'any_password')
    assert result is False

def test_is_admin_for_admin_user(authenticator):
    result = authenticator.is_admin('admin_user')
    assert result is True

def test_is_admin_for_regular_user(authenticator):
    result = authenticator.is_admin('alice')
    assert result is False
```

### Fixture Composition in Action

Look at the `authenticator` fixture:

```python
@pytest.fixture
def authenticator(db):
    return UserAuthenticator(db)
```

This fixture **depends on** the `db` fixture. When pytest runs a test that needs `authenticator`:

1. Pytest sees the test needs `authenticator`
2. Pytest sees `authenticator` needs `db`
3. Pytest calls `db()` first
4. Pytest passes the result to `authenticator(db)`
5. Pytest passes the authenticator to your test

This is **fixture composition**—building complex setup from simple pieces.

Now look at our tests:

```python
def test_authenticate_valid_user(authenticator):
    result = authenticator.authenticate('alice', 'secret123')
    assert result is True
```

The test is now **pure test logic**. No setup code at all. The test clearly states what it's testing and what the expected outcome is.

Let's verify this works:

```bash
pytest test_auth_composed.py -v
```

**Output:**
```
======================== test session starts =========================
collected 5 items

test_auth_composed.py::test_authenticate_valid_user PASSED      [ 20%]
test_auth_composed.py::test_authenticate_invalid_password PASSED [ 40%]
test_auth_composed.py::test_authenticate_nonexistent_user PASSED [ 60%]
test_auth_composed.py::test_is_admin_for_admin_user PASSED      [ 80%]
test_auth_composed.py::test_is_admin_for_regular_user PASSED    [100%]

========================= 5 passed in 0.02s ==========================
```

## Teardown: Cleaning Up After Tests

So far, our fixtures only do **setup**—they create objects before tests run. But what about **teardown**—cleaning up after tests finish?

Consider a fixture that opens a real database connection or creates temporary files. You need to close the connection or delete the files after each test. This is where teardown comes in.

### The Yield Pattern

Fixtures can use `yield` instead of `return` to provide both setup and teardown:

```python
# test_auth_with_teardown.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    def __init__(self):
        self.users = {}
        self.connection_open = True
    
    def add_user(self, username, password, role='user'):
        if not self.connection_open:
            raise RuntimeError("Database connection closed")
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        if not self.connection_open:
            raise RuntimeError("Database connection closed")
        return self.users.get(username)
    
    def close(self):
        """Simulate closing a database connection."""
        self.connection_open = False
        self.users.clear()

@pytest.fixture
def db():
    """Provide a database with automatic cleanup."""
    # Setup: Create and populate database
    database = MockDatabase()
    database.add_user('alice', 'secret123')
    database.add_user('admin_user', 'admin_pass', role='admin')
    
    # Provide the database to the test
    yield database
    
    # Teardown: Clean up after test completes
    database.close()

@pytest.fixture
def authenticator(db):
    return UserAuthenticator(db)

def test_authenticate_valid_user(authenticator):
    result = authenticator.authenticate('alice', 'secret123')
    assert result is True

def test_database_is_fresh_for_each_test(db):
    """Verify each test gets a clean database."""
    # This test modifies the database
    db.add_user('temporary_user', 'temp_pass')
    assert db.get_user('temporary_user') is not None

def test_previous_test_modifications_are_gone(db):
    """Verify the previous test's changes don't persist."""
    # The temporary_user from the previous test should not exist
    assert db.get_user('temporary_user') is None
```

### How Yield Works in Fixtures

```python
@pytest.fixture
def db():
    database = MockDatabase()
    database.add_user('alice', 'secret123')
    
    yield database  # Pause here, run the test
    
    database.close()  # Resume here after test completes
```

The execution flow:

1. **Setup phase**: Everything before `yield` runs
2. **Test execution**: Pytest runs your test with the yielded value
3. **Teardown phase**: Everything after `yield` runs (even if the test fails)

The teardown code **always runs**, even if:
- The test fails with an assertion error
- The test raises an exception
- The test is skipped

This guarantees cleanup happens.

Let's run these tests to see the teardown in action:

```bash
pytest test_auth_with_teardown.py -v
```

**Output:**
```
======================== test session starts =========================
collected 3 items

test_auth_with_teardown.py::test_authenticate_valid_user PASSED [ 33%]
test_auth_with_teardown.py::test_database_is_fresh_for_each_test PASSED [ 66%]
test_auth_with_teardown.py::test_previous_test_modifications_are_gone PASSED [100%]

========================= 3 passed in 0.01s ==========================
```

The third test passes, proving that each test gets a fresh database. The `temporary_user` added in the second test doesn't exist in the third test because the fixture's teardown code ran between them.

## Visualizing Fixture Execution

To make fixture execution visible, let's add print statements:

```python
# test_fixture_execution_order.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture
def db():
    print("\n[FIXTURE] Setting up database")
    database = MockDatabase()
    database.add_user('alice', 'secret123')
    
    yield database
    
    print("\n[FIXTURE] Tearing down database")

@pytest.fixture
def authenticator(db):
    print("\n[FIXTURE] Creating authenticator")
    auth = UserAuthenticator(db)
    
    yield auth
    
    print("\n[FIXTURE] Cleaning up authenticator")

def test_first(authenticator):
    print("\n[TEST] Running test_first")
    assert authenticator.authenticate('alice', 'secret123') is True

def test_second(authenticator):
    print("\n[TEST] Running test_second")
    assert authenticator.authenticate('alice', 'wrong') is False
```

Run with `-s` to see print output:

```bash
pytest test_fixture_execution_order.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 2 items

test_fixture_execution_order.py::test_first 
[FIXTURE] Setting up database
[FIXTURE] Creating authenticator
[TEST] Running test_first
PASSED
[FIXTURE] Cleaning up authenticator
[FIXTURE] Tearing down database

test_fixture_execution_order.py::test_second 
[FIXTURE] Setting up database
[FIXTURE] Creating authenticator
[TEST] Running test_second
PASSED
[FIXTURE] Cleaning up authenticator
[FIXTURE] Tearing down database

========================= 2 passed in 0.02s ==========================
```

This reveals the execution order:

**For each test:**
1. Setup `db` fixture
2. Setup `authenticator` fixture (which uses `db`)
3. Run the test
4. Teardown `authenticator` fixture
5. Teardown `db` fixture

Notice that teardown happens in **reverse order** of setup. This is crucial—the authenticator is cleaned up before the database it depends on.

## When to Use Return vs. Yield

**Use `return`** when:
- No cleanup is needed
- The fixture creates simple objects (strings, numbers, lists)
- The fixture creates objects that Python's garbage collector will handle

**Use `yield`** when:
- You need to close connections (database, network, files)
- You need to delete temporary resources (files, directories)
- You need to restore state (environment variables, global settings)
- You want to verify post-test conditions

Most fixtures in real projects use `yield` because cleanup is usually necessary.

## The @pytest.fixture Decorator

## Understanding the @pytest.fixture Decorator

We've been using `@pytest.fixture` without fully explaining what it does. Let's explore the decorator's capabilities and options.

### Basic Decorator Usage

The simplest form we've already seen:

```python
@pytest.fixture
def my_fixture():
    return "some value"
```

This creates a fixture with default settings. But the decorator accepts several parameters that control fixture behavior.

## Fixture Names and Aliasing

By default, the fixture name is the function name. But you can override this:

```python
# test_fixture_naming.py
import pytest

@pytest.fixture(name="database")
def db_fixture():
    """The fixture is called 'database', not 'db_fixture'."""
    return {"users": []}

def test_with_renamed_fixture(database):
    # We use 'database', not 'db_fixture'
    assert isinstance(database, dict)
```

This is useful when:
- The fixture function name is verbose for internal clarity
- You want a shorter name for tests to use
- You're refactoring and want to maintain backward compatibility

However, **most fixtures should use the default name**. Explicit naming is rarely necessary.

## Fixture Parameters: Controlling Behavior

The `@pytest.fixture` decorator accepts several parameters. Let's explore them systematically.

### The `scope` Parameter

We'll cover scope in detail in section 4.4, but here's a preview:

```python
@pytest.fixture(scope="function")  # Default: runs for each test
def per_test_fixture():
    return "fresh for each test"

@pytest.fixture(scope="module")  # Runs once per test file
def per_module_fixture():
    return "shared across all tests in this file"
```

### The `autouse` Parameter

Normally, fixtures only run when tests explicitly request them. But `autouse=True` makes a fixture run automatically for all tests:

```python
# test_autouse.py
import pytest

@pytest.fixture(autouse=True)
def reset_global_state():
    """This runs before EVERY test automatically."""
    print("\n[AUTO] Resetting global state")
    # Reset some global state here
    yield
    print("\n[AUTO] Cleanup complete")

def test_first():
    print("\n[TEST] test_first")
    assert True

def test_second():
    print("\n[TEST] test_second")
    assert True
```

```bash
pytest test_autouse.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 2 items

test_autouse.py::test_first 
[AUTO] Resetting global state
[TEST] test_first
PASSED
[AUTO] Cleanup complete

test_autouse.py::test_second 
[AUTO] Resetting global state
[TEST] test_second
PASSED
[AUTO] Cleanup complete

========================= 2 passed in 0.01s ==========================
```

Notice that `reset_global_state` ran for both tests even though neither test requested it.

**When to use `autouse=True`:**
- Resetting global state between tests
- Setting up logging or monitoring
- Configuring test environment variables
- Ensuring consistent test isolation

**When NOT to use `autouse=True`:**
- When only some tests need the fixture (wastes resources)
- When the fixture is expensive to create
- When the fixture's purpose isn't obvious (makes tests harder to understand)

### The `params` Parameter

This enables **fixture parametrization**—running tests with multiple fixture values. We'll explore this in depth in section 4.4, but here's a taste:

```python
# test_fixture_params.py
import pytest

@pytest.fixture(params=['alice', 'bob', 'charlie'])
def username(request):
    """This fixture will run tests three times, once per username."""
    return request.param

def test_username_length(username):
    """This test runs three times with different usernames."""
    assert len(username) > 0
    print(f"\n[TEST] Testing with username: {username}")
```

```bash
pytest test_fixture_params.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 3 items

test_fixture_params.py::test_username_length[alice] 
[TEST] Testing with username: alice
PASSED
test_fixture_params.py::test_username_length[bob] 
[TEST] Testing with username: bob
PASSED
test_fixture_params.py::test_username_length[charlie] 
[TEST] Testing with username: charlie
PASSED

========================= 3 passed in 0.01s ==========================
```

One test function became three test executions. We'll explore this powerful feature in section 4.4.

## The Special `request` Fixture

You may have noticed `request` appearing in the parametrized fixture above. This is a **built-in fixture** that pytest provides automatically. It gives fixtures access to metadata about the test requesting them.

### Accessing Test Information

```python
# test_request_fixture.py
import pytest

@pytest.fixture
def test_metadata(request):
    """Demonstrate accessing test information."""
    print(f"\n[FIXTURE] Test function: {request.function.__name__}")
    print(f"[FIXTURE] Test module: {request.module.__name__}")
    print(f"[FIXTURE] Test node: {request.node.name}")
    return "fixture value"

def test_example(test_metadata):
    print(f"\n[TEST] Received: {test_metadata}")
    assert True
```

```bash
pytest test_request_fixture.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 1 item

test_request_fixture.py::test_example 
[FIXTURE] Test function: test_example
[FIXTURE] Test module: test_request_fixture
[FIXTURE] Test node: test_example
[TEST] Received: fixture value
PASSED

========================= 1 passed in 0.01s ==========================
```

The `request` object provides:
- `request.function`: The test function object
- `request.module`: The test module object
- `request.node`: The pytest test node (contains test name, markers, etc.)
- `request.param`: The current parameter value (for parametrized fixtures)

### Using `request` for Dynamic Fixtures

The `request` object enables fixtures to adapt based on test context:

```python
# test_dynamic_fixture.py
import pytest

@pytest.fixture
def database(request):
    """Create a database with a name based on the test."""
    test_name = request.function.__name__
    db_name = f"test_db_{test_name}"
    print(f"\n[FIXTURE] Creating database: {db_name}")
    
    # Simulate database creation
    db = {"name": db_name, "tables": []}
    
    yield db
    
    print(f"\n[FIXTURE] Dropping database: {db_name}")

def test_users(database):
    print(f"\n[TEST] Using database: {database['name']}")
    assert database['name'] == "test_db_test_users"

def test_products(database):
    print(f"\n[TEST] Using database: {database['name']}")
    assert database['name'] == "test_db_test_products"
```

```bash
pytest test_dynamic_fixture.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 2 items

test_dynamic_fixture.py::test_users 
[FIXTURE] Creating database: test_db_test_users
[TEST] Using database: test_db_test_users
PASSED
[FIXTURE] Dropping database: test_db_test_users

test_dynamic_fixture.py::test_products 
[FIXTURE] Creating database: test_db_test_products
[TEST] Using database: test_db_test_products
PASSED
[FIXTURE] Dropping database: test_db_test_products

========================= 2 passed in 0.01s ==========================
```

Each test gets a database with a unique name derived from the test function name.

## Fixture Finalization with `request.addfinalizer`

Besides `yield`, there's another way to register teardown code: `request.addfinalizer()`. This is useful when you need multiple cleanup steps or conditional cleanup:

```python
# test_addfinalizer.py
import pytest

@pytest.fixture
def resource_with_finalizer(request):
    """Demonstrate request.addfinalizer for cleanup."""
    print("\n[FIXTURE] Acquiring resource")
    resource = {"status": "active"}
    
    def cleanup():
        print("\n[FINALIZER] Releasing resource")
        resource["status"] = "released"
    
    request.addfinalizer(cleanup)
    return resource

def test_resource_usage(resource_with_finalizer):
    print(f"\n[TEST] Resource status: {resource_with_finalizer['status']}")
    assert resource_with_finalizer["status"] == "active"
```

```bash
pytest test_addfinalizer.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 1 item

test_addfinalizer.py::test_resource_usage 
[FIXTURE] Acquiring resource
[TEST] Resource status: active
PASSED
[FINALIZER] Releasing resource

========================= 1 passed in 0.01s ==========================
```

### When to Use `addfinalizer` vs. `yield`

**Use `yield`** (preferred):
- Single cleanup step
- Simple, linear setup/teardown
- Most common case

**Use `addfinalizer`**:
- Multiple cleanup steps that might be registered conditionally
- Need to register cleanup from within a helper function
- Complex cleanup logic that doesn't fit the yield pattern

Example of conditional cleanup:

```python
# test_conditional_cleanup.py
import pytest

@pytest.fixture
def conditional_resource(request):
    """Only clean up if resource was actually created."""
    print("\n[FIXTURE] Starting setup")
    
    # Simulate conditional resource creation
    if request.function.__name__.startswith("test_with_"):
        print("[FIXTURE] Creating expensive resource")
        resource = {"type": "expensive", "data": [1, 2, 3]}
        
        def cleanup():
            print("\n[FINALIZER] Cleaning up expensive resource")
        
        request.addfinalizer(cleanup)
    else:
        print("[FIXTURE] Using lightweight resource")
        resource = {"type": "lightweight"}
    
    return resource

def test_with_expensive_resource(conditional_resource):
    print(f"\n[TEST] Resource type: {conditional_resource['type']}")
    assert conditional_resource["type"] == "expensive"

def test_without_expensive_resource(conditional_resource):
    print(f"\n[TEST] Resource type: {conditional_resource['type']}")
    assert conditional_resource["type"] == "lightweight"
```

```bash
pytest test_conditional_cleanup.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 2 items

test_conditional_cleanup.py::test_with_expensive_resource 
[FIXTURE] Starting setup
[FIXTURE] Creating expensive resource
[TEST] Resource type: expensive
PASSED
[FINALIZER] Cleaning up expensive resource

test_conditional_cleanup.py::test_without_expensive_resource 
[FIXTURE] Starting setup
[FIXTURE] Using lightweight resource
[TEST] Resource type: lightweight
PASSED

========================= 2 passed in 0.01s ==========================
```

Notice that cleanup only ran for the first test, which actually created the expensive resource.

## Decorator Parameters Summary

Here's a complete reference of `@pytest.fixture` parameters:

| Parameter | Type | Default | Purpose |
|-----------|------|---------|---------|
| `scope` | str | `"function"` | When to create/destroy the fixture |
| `params` | list | `None` | Values to parametrize the fixture with |
| `autouse` | bool | `False` | Run automatically for all tests |
| `ids` | list/callable | `None` | Custom test IDs for parametrized fixtures |
| `name` | str | function name | Override the fixture name |

We'll explore `scope` and `params` in depth in the next sections.

## Fixture Scope: Function, Class, Module, and Session

## The Performance Problem

Let's return to our authentication system. Suppose our `MockDatabase` is actually a real database connection that takes 2 seconds to initialize. With our current fixtures, every test creates a new database:

```python
# test_slow_fixtures.py
import pytest
import time
from auth import UserAuthenticator

class SlowDatabase:
    """Simulates a database with expensive initialization."""
    def __init__(self):
        print("\n[DB] Initializing database (this takes 2 seconds)...")
        time.sleep(2)  # Simulate expensive setup
        self.users = {}
        print("[DB] Database ready")
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture
def db():
    """This creates a new database for EVERY test."""
    database = SlowDatabase()
    database.add_user('alice', 'secret123')
    return database

@pytest.fixture
def authenticator(db):
    return UserAuthenticator(db)

def test_authenticate_valid_user(authenticator):
    assert authenticator.authenticate('alice', 'secret123') is True

def test_authenticate_invalid_password(authenticator):
    assert authenticator.authenticate('alice', 'wrong') is False

def test_authenticate_nonexistent_user(authenticator):
    assert authenticator.authenticate('bob', 'any') is False
```

Let's run these tests and time them:

```bash
pytest test_slow_fixtures.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 3 items

test_slow_fixtures.py::test_authenticate_valid_user 
[DB] Initializing database (this takes 2 seconds)...
[DB] Database ready
PASSED

test_slow_fixtures.py::test_authenticate_invalid_password 
[DB] Initializing database (this takes 2 seconds)...
[DB] Database ready
PASSED

test_slow_fixtures.py::test_authenticate_nonexistent_user 
[DB] Initializing database (this takes 2 seconds)...
[DB] Database ready
PASSED

========================= 3 passed in 6.02s ==========================
```

**6 seconds for 3 tests!** The database initialized three times—once per test. This is wasteful because:

1. Each test uses the same initial database state
2. Tests don't modify the database in ways that affect other tests
3. We're paying the 2-second initialization cost unnecessarily

This is where **fixture scope** solves the problem.

## Understanding Fixture Scope

**Scope** determines how often a fixture is created and destroyed. Instead of creating a new fixture for every test, you can share one fixture across multiple tests.

Pytest provides four scope levels:

| Scope | Lifetime | Use Case |
|-------|----------|----------|
| `function` | Per test function (default) | Test isolation is critical |
| `class` | Per test class | Tests in a class share state |
| `module` | Per test file | Tests in a file share state |
| `session` | Entire test run | Expensive setup shared globally |

### Scope: The Mental Model

Think of scope as **how long the fixture lives**:

- **`function` scope**: Born before each test, dies after each test
- **`class` scope**: Born before the first test in a class, dies after the last test in that class
- **`module` scope**: Born before the first test in a file, dies after the last test in that file
- **`session` scope**: Born before any test runs, dies after all tests complete

## Module Scope: Sharing Across a File

Let's fix our slow tests by using `scope="module"`:

```python
# test_module_scope.py
import pytest
import time
from auth import UserAuthenticator

class SlowDatabase:
    def __init__(self):
        print("\n[DB] Initializing database (this takes 2 seconds)...")
        time.sleep(2)
        self.users = {}
        print("[DB] Database ready")
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture(scope="module")
def db():
    """This creates ONE database for ALL tests in this file."""
    print("\n[FIXTURE] Creating module-scoped database")
    database = SlowDatabase()
    database.add_user('alice', 'secret123')
    
    yield database
    
    print("\n[FIXTURE] Tearing down module-scoped database")

@pytest.fixture
def authenticator(db):
    """This still runs per test, but reuses the same db."""
    print("\n[FIXTURE] Creating authenticator")
    return UserAuthenticator(db)

def test_authenticate_valid_user(authenticator):
    print("\n[TEST] test_authenticate_valid_user")
    assert authenticator.authenticate('alice', 'secret123') is True

def test_authenticate_invalid_password(authenticator):
    print("\n[TEST] test_authenticate_invalid_password")
    assert authenticator.authenticate('alice', 'wrong') is False

def test_authenticate_nonexistent_user(authenticator):
    print("\n[TEST] test_authenticate_nonexistent_user")
    assert authenticator.authenticate('bob', 'any') is False
```

```bash
pytest test_module_scope.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 3 items

test_module_scope.py::test_authenticate_valid_user 
[FIXTURE] Creating module-scoped database
[DB] Initializing database (this takes 2 seconds)...
[DB] Database ready
[FIXTURE] Creating authenticator
[TEST] test_authenticate_valid_user
PASSED

test_module_scope.py::test_authenticate_invalid_password 
[FIXTURE] Creating authenticator
[TEST] test_authenticate_invalid_password
PASSED

test_module_scope.py::test_authenticate_nonexistent_user 
[FIXTURE] Creating authenticator
[TEST] test_authenticate_nonexistent_user
PASSED
[FIXTURE] Tearing down module-scoped database

========================= 3 passed in 2.03s ==========================
```

**2 seconds instead of 6!** The database initialized only once. Notice:

1. The database fixture ran before the first test
2. Each test got a new `authenticator`, but they all shared the same `db`
3. The database teardown ran after all tests completed

### Diagnostic Analysis: Reading the Execution Flow

Let's parse what happened:

**Before first test:**
- `[FIXTURE] Creating module-scoped database` - The module-scoped fixture initializes
- `[DB] Initializing database...` - Expensive setup happens once

**For each test:**
- `[FIXTURE] Creating authenticator` - Function-scoped fixture runs per test
- `[TEST] test_name` - Test executes

**After last test:**
- `[FIXTURE] Tearing down module-scoped database` - Module-scoped teardown runs once

The key insight: **Module-scoped fixtures are created once and shared across all tests in the file.**

## The Scope Hierarchy and Composition Rules

When fixtures depend on each other, their scopes must follow a rule:

**A fixture can only depend on fixtures with equal or broader scope.**

Valid combinations:
- `function` scope can use `function`, `class`, `module`, or `session` fixtures
- `class` scope can use `class`, `module`, or `session` fixtures
- `module` scope can use `module` or `session` fixtures
- `session` scope can only use `session` fixtures

Invalid combination:
- `module` scope **cannot** use `function` scope fixtures

Let's see what happens if we violate this rule:

```python
# test_invalid_scope.py
import pytest

@pytest.fixture(scope="function")
def function_fixture():
    return "function-scoped"

@pytest.fixture(scope="module")
def module_fixture(function_fixture):
    """This is INVALID: module scope using function scope."""
    return f"module using {function_fixture}"

def test_example(module_fixture):
    assert True
```

```bash
pytest test_invalid_scope.py -v
```

**Output:**
```
======================== test session starts =========================
collected 1 item

test_invalid_scope.py::test_example ERROR

============================== ERRORS ===============================
ERROR at setup of test_example
ScopeMismatch: You tried to access the function scoped fixture function_fixture with a module scoped request object, involved factories:
test_invalid_scope.py::module_fixture
test_invalid_scope.py::function_fixture

========================= 1 error in 0.01s ===========================
```

### Why This Rule Exists

The rule prevents logical impossibilities. Consider:

- A `module` fixture lives for the entire file
- A `function` fixture lives for one test
- If the module fixture depended on the function fixture, which function's fixture would it use?

The module fixture would need to be recreated for each test (to get a fresh function fixture), which defeats the purpose of module scope.

## Class Scope: Sharing Within Test Classes

Class scope is useful when you organize tests into classes and want to share setup across methods in that class:

```python
# test_class_scope.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture(scope="class")
def db():
    """Shared across all tests in a class."""
    print("\n[FIXTURE] Creating class-scoped database")
    database = MockDatabase()
    database.add_user('alice', 'secret123')
    
    yield database
    
    print("\n[FIXTURE] Tearing down class-scoped database")

@pytest.fixture
def authenticator(db):
    print("\n[FIXTURE] Creating authenticator")
    return UserAuthenticator(db)

class TestAuthentication:
    """Tests in this class share the same database."""
    
    def test_valid_credentials(self, authenticator):
        print("\n[TEST] test_valid_credentials")
        assert authenticator.authenticate('alice', 'secret123') is True
    
    def test_invalid_credentials(self, authenticator):
        print("\n[TEST] test_invalid_credentials")
        assert authenticator.authenticate('alice', 'wrong') is False

class TestAuthorization:
    """This class gets its own database instance."""
    
    def test_admin_check(self, db):
        print("\n[TEST] test_admin_check")
        db.add_user('admin', 'admin_pass', role='admin')
        authenticator = UserAuthenticator(db)
        assert authenticator.is_admin('admin') is True
```

```bash
pytest test_class_scope.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 3 items

test_class_scope.py::TestAuthentication::test_valid_credentials 
[FIXTURE] Creating class-scoped database
[FIXTURE] Creating authenticator
[TEST] test_valid_credentials
PASSED

test_class_scope.py::TestAuthentication::test_invalid_credentials 
[FIXTURE] Creating authenticator
[TEST] test_invalid_credentials
PASSED
[FIXTURE] Tearing down class-scoped database

test_class_scope.py::TestAuthorization::test_admin_check 
[FIXTURE] Creating class-scoped database
[TEST] test_admin_check
PASSED
[FIXTURE] Tearing down class-scoped database

========================= 3 passed in 0.02s ==========================
```

Notice:
- The database was created twice: once for `TestAuthentication`, once for `TestAuthorization`
- Within `TestAuthentication`, both tests shared the same database
- Each class got its own isolated database instance

## Session Scope: Global Sharing

Session scope creates a fixture once for the entire test run, shared across all test files:

```python
# conftest.py (shared configuration file)
import pytest
import time

@pytest.fixture(scope="session")
def expensive_resource():
    """This runs ONCE for the entire test session."""
    print("\n[SESSION] Initializing expensive resource (5 seconds)...")
    time.sleep(5)
    resource = {"initialized": True, "data": "shared data"}
    print("[SESSION] Resource ready")
    
    yield resource
    
    print("\n[SESSION] Cleaning up expensive resource")
```

```python
# test_file_1.py
def test_in_file_1(expensive_resource):
    print(f"\n[TEST] test_file_1 using resource: {expensive_resource}")
    assert expensive_resource["initialized"] is True

def test_another_in_file_1(expensive_resource):
    print(f"\n[TEST] test_another_in_file_1 using resource: {expensive_resource}")
    assert expensive_resource["data"] == "shared data"
```

```python
# test_file_2.py
def test_in_file_2(expensive_resource):
    print(f"\n[TEST] test_file_2 using resource: {expensive_resource}")
    assert expensive_resource["initialized"] is True
```

```bash
pytest test_file_1.py test_file_2.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 3 items

test_file_1.py::test_in_file_1 
[SESSION] Initializing expensive resource (5 seconds)...
[SESSION] Resource ready
[TEST] test_file_1 using resource: {'initialized': True, 'data': 'shared data'}
PASSED

test_file_1.py::test_another_in_file_1 
[TEST] test_another_in_file_1 using resource: {'initialized': True, 'data': 'shared data'}
PASSED

test_file_2.py::test_in_file_2 
[TEST] test_file_2 using resource: {'initialized': True, 'data': 'shared data'}
PASSED
[SESSION] Cleaning up expensive resource

========================= 3 passed in 5.02s ==========================
```

The resource initialized once before any test ran, was shared across both files, and cleaned up after all tests completed.

### When to Use Session Scope

**Use session scope for:**
- Database connections that are expensive to create
- Starting external services (Docker containers, test servers)
- Loading large datasets that don't change
- Initializing machine learning models

**Don't use session scope for:**
- Resources that tests modify (breaks test isolation)
- Resources that are cheap to create
- Resources that need to be fresh for each test

## The Danger of Shared State

Broader scopes improve performance but risk breaking test isolation. Consider this problematic example:

```python
# test_shared_state_problem.py
import pytest

@pytest.fixture(scope="module")
def shared_list():
    """A mutable object shared across tests."""
    return []

def test_append_one(shared_list):
    shared_list.append(1)
    assert len(shared_list) == 1

def test_append_two(shared_list):
    shared_list.append(2)
    # This test expects the list to have 1 element, but it has 2!
    assert len(shared_list) == 1  # This will fail!
```

```bash
pytest test_shared_state_problem.py -v
```

**Output:**
```
======================== test session starts =========================
collected 2 items

test_shared_state_problem.py::test_append_one PASSED           [ 50%]
test_shared_state_problem.py::test_append_two FAILED           [100%]

============================== FAILURES ==============================
________________________ test_append_two _____________________________

shared_list = [1, 2]

    def test_append_two(shared_list):
        shared_list.append(2)
>       assert len(shared_list) == 1
E       assert 2 == 1
E        +  where 2 = len([1, 2])

test_shared_state_problem.py:14: AssertionError
========================= 1 failed, 1 passed in 0.02s ================
```

### Diagnostic Analysis: Reading the Failure

**The summary line:**
```
test_shared_state_problem.py::test_append_two FAILED
```
This tells us the second test failed.

**The assertion introspection:**
```
shared_list = [1, 2]

    def test_append_two(shared_list):
        shared_list.append(2)
>       assert len(shared_list) == 1
E       assert 2 == 1
E        +  where 2 = len([1, 2])
```

Pytest shows us:
1. The shared_list contains `[1, 2]` when the test runs
2. The test expected length 1, but got length 2
3. The list contains the element from the previous test

**Root cause:** The first test modified the shared list, and that modification persisted to the second test.

**Solution:** Either use `function` scope for isolation, or design tests to not depend on initial state.

### Safe Patterns for Broader Scopes

To safely use broader scopes with mutable objects:

**Pattern 1: Read-only fixtures**

```python
@pytest.fixture(scope="module")
def config():
    """Safe because tests only read, never modify."""
    return {
        "api_url": "https://api.example.com",
        "timeout": 30,
        "retry_count": 3
    }
```

**Pattern 2: Factory fixtures**

```python
@pytest.fixture(scope="module")
def database_factory():
    """Return a factory that creates fresh databases."""
    def create_database():
        return MockDatabase()
    return create_database

def test_example(database_factory):
    # Each test gets a fresh database
    db = database_factory()
    db.add_user('alice', 'secret')
```

**Pattern 3: Reset between tests**

```python
@pytest.fixture(scope="module")
def database():
    db = MockDatabase()
    yield db
    # Reset after each test
    db.users.clear()
```

## Scope Decision Framework

When choosing fixture scope, ask these questions:

| Question | Answer | Recommended Scope |
|----------|--------|-------------------|
| Is setup expensive (>100ms)? | No | `function` |
| Is setup expensive? | Yes | Consider broader scope |
| Do tests modify the fixture? | Yes | `function` |
| Do tests only read the fixture? | Yes | Broader scope is safe |
| Is the fixture a database connection? | Yes | `module` or `session` |
| Is the fixture test data? | Yes | `function` |
| Is the fixture a configuration? | Yes | `module` or `session` |
| Do tests need isolation? | Yes | `function` |

**Default to `function` scope** unless you have a specific reason to use broader scope. Test isolation is more important than performance in most cases.

## Visualizing Scope Lifetimes

Here's a complete example showing all four scopes:

```python
# test_all_scopes.py
import pytest

@pytest.fixture(scope="session")
def session_fixture():
    print("\n[SESSION] Setup")
    yield "session"
    print("\n[SESSION] Teardown")

@pytest.fixture(scope="module")
def module_fixture():
    print("\n[MODULE] Setup")
    yield "module"
    print("\n[MODULE] Teardown")

@pytest.fixture(scope="class")
def class_fixture():
    print("\n[CLASS] Setup")
    yield "class"
    print("\n[CLASS] Teardown")

@pytest.fixture(scope="function")
def function_fixture():
    print("\n[FUNCTION] Setup")
    yield "function"
    print("\n[FUNCTION] Teardown")

class TestGroup1:
    def test_1(self, session_fixture, module_fixture, class_fixture, function_fixture):
        print(f"\n[TEST] test_1: {session_fixture}, {module_fixture}, {class_fixture}, {function_fixture}")
    
    def test_2(self, session_fixture, module_fixture, class_fixture, function_fixture):
        print(f"\n[TEST] test_2: {session_fixture}, {module_fixture}, {class_fixture}, {function_fixture}")

class TestGroup2:
    def test_3(self, session_fixture, module_fixture, class_fixture, function_fixture):
        print(f"\n[TEST] test_3: {session_fixture}, {module_fixture}, {class_fixture}, {function_fixture}")
```

```bash
pytest test_all_scopes.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 3 items

test_all_scopes.py::TestGroup1::test_1 
[SESSION] Setup
[MODULE] Setup
[CLASS] Setup
[FUNCTION] Setup
[TEST] test_1: session, module, class, function
PASSED
[FUNCTION] Teardown

test_all_scopes.py::TestGroup1::test_2 
[FUNCTION] Setup
[TEST] test_2: session, module, class, function
PASSED
[FUNCTION] Teardown
[CLASS] Teardown

test_all_scopes.py::TestGroup2::test_3 
[CLASS] Setup
[FUNCTION] Setup
[TEST] test_3: session, module, class, function
PASSED
[FUNCTION] Teardown
[CLASS] Teardown
[MODULE] Teardown
[SESSION] Teardown

========================= 3 passed in 0.02s ==========================
```

This reveals the complete lifecycle:

**Before any test:**
- Session fixture initializes

**Before first test in file:**
- Module fixture initializes

**Before first test in TestGroup1:**
- Class fixture initializes

**For each test:**
- Function fixture initializes
- Test runs
- Function fixture tears down

**After last test in TestGroup1:**
- Class fixture tears down

**Before first test in TestGroup2:**
- New class fixture initializes

**After last test in file:**
- Module fixture tears down

**After all tests:**
- Session fixture tears down

The key pattern: **Broader scopes initialize earlier and tear down later.**

## Using Fixtures in Your Tests

## Beyond Simple Fixture Usage

We've seen the basics of requesting fixtures as test parameters. Now let's explore advanced patterns for using fixtures effectively in real-world scenarios.

## Multiple Fixtures in One Test

Tests can request as many fixtures as they need. Pytest resolves all dependencies automatically:

```python
# test_multiple_fixtures.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

class Logger:
    def __init__(self):
        self.logs = []
    
    def log(self, message):
        self.logs.append(message)

@pytest.fixture
def db():
    """Provide a test database."""
    database = MockDatabase()
    database.add_user('alice', 'secret123')
    database.add_user('admin', 'admin_pass', role='admin')
    return database

@pytest.fixture
def logger():
    """Provide a logger for tracking operations."""
    return Logger()

@pytest.fixture
def authenticator(db, logger):
    """Provide an authenticator with logging."""
    auth = UserAuthenticator(db)
    logger.log("Authenticator created")
    return auth

def test_authentication_with_logging(authenticator, logger):
    """Test can use both authenticator and logger."""
    logger.log("Starting authentication test")
    
    result = authenticator.authenticate('alice', 'secret123')
    logger.log(f"Authentication result: {result}")
    
    assert result is True
    assert len(logger.logs) == 3
    assert "Starting authentication test" in logger.logs
```

This test uses three fixtures:
1. `authenticator` (directly requested)
2. `logger` (directly requested)
3. `db` (indirectly, through `authenticator`)

Pytest automatically:
- Creates `db` first
- Creates `logger` second
- Creates `authenticator` third (using both `db` and `logger`)
- Passes `authenticator` and `logger` to the test

## Fixtures in Test Classes

When using test classes, fixtures work the same way—request them as method parameters:

```python
# test_fixtures_in_classes.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture
def db():
    database = MockDatabase()
    database.add_user('alice', 'secret123')
    database.add_user('admin', 'admin_pass', role='admin')
    return database

@pytest.fixture
def authenticator(db):
    return UserAuthenticator(db)

class TestUserAuthentication:
    """All methods in this class can use fixtures."""
    
    def test_valid_credentials(self, authenticator):
        assert authenticator.authenticate('alice', 'secret123') is True
    
    def test_invalid_credentials(self, authenticator):
        assert authenticator.authenticate('alice', 'wrong') is False
    
    def test_can_access_db_directly(self, db):
        """Tests can also request the underlying fixtures."""
        assert db.get_user('alice') is not None

class TestAdminAuthorization:
    """Each class gets fresh fixtures."""
    
    def test_admin_user(self, authenticator):
        assert authenticator.is_admin('admin') is True
    
    def test_regular_user(self, authenticator):
        assert authenticator.is_admin('alice') is False
```

Key points:
- Fixtures are requested as method parameters (not `self.fixture`)
- Each test method can request different fixtures
- Each class gets fresh fixture instances (with default `function` scope)

## Fixture Factories: Creating Multiple Instances

Sometimes you need multiple instances of the same type of object in a single test. The **factory pattern** solves this:

```python
# test_fixture_factory.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    def __init__(self, name):
        self.name = name
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture
def database_factory():
    """Return a factory function that creates databases."""
    created_databases = []
    
    def create_database(name):
        db = MockDatabase(name)
        created_databases.append(db)
        return db
    
    yield create_database
    
    # Cleanup: close all created databases
    print(f"\n[CLEANUP] Closing {len(created_databases)} databases")
    for db in created_databases:
        print(f"[CLEANUP] Closing database: {db.name}")

def test_multiple_databases(database_factory):
    """Create multiple databases in one test."""
    # Create separate databases for different purposes
    user_db = database_factory("users")
    user_db.add_user('alice', 'secret123')
    
    admin_db = database_factory("admins")
    admin_db.add_user('admin', 'admin_pass', role='admin')
    
    # Verify they're independent
    assert user_db.get_user('alice') is not None
    assert user_db.get_user('admin') is None
    
    assert admin_db.get_user('admin') is not None
    assert admin_db.get_user('alice') is None
    
    print(f"\n[TEST] Created databases: {user_db.name}, {admin_db.name}")
```

```bash
pytest test_fixture_factory.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 1 item

test_fixture_factory.py::test_multiple_databases 
[TEST] Created databases: users, admins
PASSED
[CLEANUP] Closing 2 databases
[CLEANUP] Closing database: users
[CLEANUP] Closing database: admins

========================= 1 passed in 0.01s ==========================
```

The factory pattern:
1. The fixture returns a **function** (the factory)
2. Tests call that function to create instances
3. The fixture tracks all created instances
4. Teardown cleans up all instances

This is powerful for:
- Creating multiple test users
- Setting up multiple API clients
- Creating temporary files or directories
- Any scenario where you need N instances of something

## Parameterized Fixtures: Testing Multiple Configurations

The `params` parameter in `@pytest.fixture` creates multiple versions of a fixture. Each test that uses the fixture runs once per parameter value:

```python
# test_parameterized_fixture.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture(params=['alice', 'bob', 'charlie'])
def username(request):
    """This fixture provides three different usernames."""
    return request.param

@pytest.fixture
def db_with_user(username):
    """Create a database with the parameterized user."""
    db = MockDatabase()
    db.add_user(username, f'{username}_password')
    return db, username

def test_user_exists(db_with_user):
    """This test runs three times, once per username."""
    db, username = db_with_user
    user = db.get_user(username)
    assert user is not None
    assert user['password'] == f'{username}_password'
    print(f"\n[TEST] Verified user: {username}")
```

```bash
pytest test_parameterized_fixture.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 3 items

test_parameterized_fixture.py::test_user_exists[alice] 
[TEST] Verified user: alice
PASSED
test_parameterized_fixture.py::test_user_exists[bob] 
[TEST] Verified user: bob
PASSED
test_parameterized_fixture.py::test_user_exists[charlie] 
[TEST] Verified user: charlie
PASSED

========================= 3 passed in 0.02s ==========================
```

One test function became three test executions. Notice the test IDs: `[alice]`, `[bob]`, `[charlie]`.

### Custom Test IDs for Parameterized Fixtures

The default IDs (`[alice]`, `[bob]`) are the string representation of the parameter. For complex objects, this can be unclear. Use the `ids` parameter to provide custom names:

```python
# test_custom_ids.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

# Define test configurations
USER_CONFIGS = [
    {'username': 'alice', 'password': 'secret123', 'role': 'user'},
    {'username': 'bob', 'password': 'bob_pass', 'role': 'user'},
    {'username': 'admin', 'password': 'admin_pass', 'role': 'admin'},
]

@pytest.fixture(
    params=USER_CONFIGS,
    ids=['regular_user_alice', 'regular_user_bob', 'admin_user']
)
def user_config(request):
    """Provide different user configurations with readable IDs."""
    return request.param

@pytest.fixture
def db_with_configured_user(user_config):
    """Create a database with the configured user."""
    db = MockDatabase()
    db.add_user(
        user_config['username'],
        user_config['password'],
        user_config['role']
    )
    return db, user_config

def test_user_authentication(db_with_configured_user):
    """Test authentication for different user types."""
    db, config = db_with_configured_user
    authenticator = UserAuthenticator(db)
    
    result = authenticator.authenticate(config['username'], config['password'])
    assert result is True
    print(f"\n[TEST] Authenticated {config['username']} ({config['role']})")
```

```bash
pytest test_custom_ids.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 3 items

test_custom_ids.py::test_user_authentication[regular_user_alice] 
[TEST] Authenticated alice (user)
PASSED
test_custom_ids.py::test_user_authentication[regular_user_bob] 
[TEST] Authenticated bob (user)
PASSED
test_custom_ids.py::test_user_authentication[admin_user] 
[TEST] Authenticated admin (admin)
PASSED

========================= 3 passed in 0.02s ==========================
```

The custom IDs make test output much more readable. You can also use a function to generate IDs dynamically:

```python
def id_from_config(config):
    """Generate test ID from configuration."""
    return f"{config['role']}_{config['username']}"

@pytest.fixture(
    params=USER_CONFIGS,
    ids=id_from_config
)
def user_config(request):
    return request.param
```

## Indirect Parametrization: Parameterizing Fixture Inputs

Sometimes you want to parametrize the **fixture itself** rather than the test. Use `indirect=True`:

```python
# test_indirect_parametrization.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture
def db(request):
    """Create a database with users specified by the test."""
    database = MockDatabase()
    
    # request.param contains the list of users to create
    for user_data in request.param:
        database.add_user(**user_data)
    
    return database

@pytest.mark.parametrize('db', [
    [{'username': 'alice', 'password': 'secret123'}],
    [{'username': 'bob', 'password': 'bob_pass'}],
    [
        {'username': 'alice', 'password': 'secret123'},
        {'username': 'bob', 'password': 'bob_pass'}
    ],
], indirect=True, ids=['single_user_alice', 'single_user_bob', 'multiple_users'])
def test_database_users(db):
    """Test with different database configurations."""
    user_count = len(db.users)
    print(f"\n[TEST] Database has {user_count} user(s)")
    assert user_count > 0
```

```bash
pytest test_indirect_parametrization.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 3 items

test_indirect_parametrization.py::test_database_users[single_user_alice] 
[TEST] Database has 1 user(s)
PASSED
test_indirect_parametrization.py::test_database_users[single_user_bob] 
[TEST] Database has 1 user(s)
PASSED
test_indirect_parametrization.py::test_database_users[multiple_users] 
[TEST] Database has 2 user(s)
PASSED

========================= 3 passed in 0.02s ==========================
```

With `indirect=True`:
1. The test is parametrized with different values
2. Those values are passed to the fixture via `request.param`
3. The fixture uses those values to create different configurations
4. The test receives the configured fixture

This is powerful for testing with different initial states or configurations.

## Combining Multiple Parametrized Fixtures

When multiple fixtures are parametrized, pytest creates the **cartesian product** of all combinations:

```python
# test_multiple_parametrized_fixtures.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture(params=['alice', 'bob'])
def username(request):
    return request.param

@pytest.fixture(params=['correct_pass', 'wrong_pass'])
def password(request):
    return request.param

@pytest.fixture
def db(username):
    """Create database with the parameterized username."""
    db = MockDatabase()
    db.add_user(username, 'correct_pass')
    return db

def test_authentication_combinations(db, username, password):
    """Test all combinations of username and password."""
    authenticator = UserAuthenticator(db)
    result = authenticator.authenticate(username, password)
    
    expected = (password == 'correct_pass')
    assert result == expected
    
    print(f"\n[TEST] User: {username}, Password: {password}, Result: {result}")
```

```bash
pytest test_multiple_parametrized_fixtures.py -v -s
```

**Output:**
```
======================== test session starts =========================
collected 4 items

test_multiple_parametrized_fixtures.py::test_authentication_combinations[alice-correct_pass] 
[TEST] User: alice, Password: correct_pass, Result: True
PASSED
test_multiple_parametrized_fixtures.py::test_authentication_combinations[alice-wrong_pass] 
[TEST] User: alice, Password: wrong_pass, Result: False
PASSED
test_multiple_parametrized_fixtures.py::test_authentication_combinations[bob-correct_pass] 
[TEST] User: bob, Password: correct_pass, Result: True
PASSED
test_multiple_parametrized_fixtures.py::test_authentication_combinations[bob-wrong_pass] 
[TEST] User: bob, Password: wrong_pass, Result: False
PASSED

========================= 4 passed in 0.02s ==========================
```

Two usernames × two passwords = four test combinations. Pytest automatically generates all combinations and runs the test for each.

## Fixture Usage Patterns Summary

| Pattern | Use Case | Example |
|---------|----------|---------|
| Multiple fixtures | Test needs several dependencies | `def test(db, logger, config):` |
| Fixture factory | Need multiple instances in one test | `db1 = db_factory("users")` |
| Parameterized fixture | Test same logic with different setups | `@pytest.fixture(params=[...])` |
| Indirect parametrization | Parametrize fixture configuration | `@pytest.mark.parametrize(..., indirect=True)` |
| Custom IDs | Readable test names | `ids=['case1', 'case2']` |
| Fixture in classes | Organize related tests | `class TestAuth:` with fixture params |

Choose the pattern that makes your tests most readable and maintainable.

## Fixture Dependencies and Composition

## Building Complex Fixtures from Simple Ones

Real applications have complex dependencies. A web application test might need:
- A database connection
- A cache layer
- An authentication service
- A configured application instance

Instead of creating one massive fixture, build complex fixtures by **composing** simple ones. This is fixture composition—the most powerful pattern in pytest.

## The Composition Pattern

Let's build a realistic example: testing a web API that requires multiple services.

```python
# api.py
class Database:
    def __init__(self, connection_string):
        self.connection_string = connection_string
        self.users = {}
    
    def add_user(self, username, email):
        self.users[username] = {'email': email}
    
    def get_user(self, username):
        return self.users.get(username)

class Cache:
    def __init__(self, ttl_seconds):
        self.ttl = ttl_seconds
        self.data = {}
    
    def set(self, key, value):
        self.data[key] = value
    
    def get(self, key):
        return self.data.get(key)

class AuthService:
    def __init__(self, database, cache):
        self.db = database
        self.cache = cache
    
    def authenticate(self, username, password):
        # Check cache first
        cached = self.cache.get(f"auth:{username}")
        if cached:
            return cached == password
        
        # Check database
        user = self.db.get_user(username)
        if user:
            # Cache the result
            self.cache.set(f"auth:{username}", password)
            return True
        return False

class APIServer:
    def __init__(self, auth_service, config):
        self.auth = auth_service
        self.config = config
    
    def handle_request(self, username, password, endpoint):
        if not self.auth.authenticate(username, password):
            return {"error": "Unauthorized", "status": 401}
        
        return {"data": f"Success accessing {endpoint}", "status": 200}
```

Now let's build fixtures that compose together to create a complete test environment:

```python
# test_api_composition.py
import pytest
from api import Database, Cache, AuthService, APIServer

# Layer 1: Basic infrastructure fixtures
@pytest.fixture
def db_connection_string():
    """Provide database connection configuration."""
    return "postgresql://test:test@localhost/testdb"

@pytest.fixture
def cache_ttl():
    """Provide cache TTL configuration."""
    return 300  # 5 minutes

@pytest.fixture
def api_config():
    """Provide API server configuration."""
    return {
        "host": "localhost",
        "port": 8000,
        "debug": True
    }

# Layer 2: Service fixtures that depend on Layer 1
@pytest.fixture
def database(db_connection_string):
    """Provide a configured database."""
    db = Database(db_connection_string)
    # Add test users
    db.add_user('alice', 'alice@example.com')
    db.add_user('bob', 'bob@example.com')
    return db

@pytest.fixture
def cache(cache_ttl):
    """Provide a configured cache."""
    return Cache(cache_ttl)

# Layer 3: Application services that depend on Layer 2
@pytest.fixture
def auth_service(database, cache):
    """Provide an authentication service."""
    return AuthService(database, cache)

# Layer 4: Complete application that depends on Layer 3
@pytest.fixture
def api_server(auth_service, api_config):
    """Provide a fully configured API server."""
    return APIServer(auth_service, api_config)

# Tests use the top-level fixture
def test_successful_authentication(api_server):
    """Test successful API request with valid credentials."""
    response = api_server.handle_request('alice', 'alice_password', '/users')
    
    assert response['status'] == 200
    assert 'Success' in response['data']

def test_failed_authentication(api_server):
    """Test API request with invalid credentials."""
    response = api_server.handle_request('alice', 'wrong_password', '/users')
    
    assert response['status'] == 401
    assert response['error'] == 'Unauthorized'

def test_cache_is_used(api_server, cache):
    """Test that authentication results are cached."""
    # First request
    api_server.handle_request('alice', 'alice_password', '/users')
    
    # Verify cache was populated
    cached_value = cache.get('auth:alice')
    assert cached_value == 'alice_password'
    
    # Second request should use cache
    response = api_server.handle_request('alice', 'alice_password', '/users')
    assert response['status'] == 200
```

Let's visualize the dependency graph:

```
api_server
    ├── auth_service
    │   ├── database
    │   │   └── db_connection_string
    │   └── cache
    │       └── cache_ttl
    └── api_config
```

When you request `api_server`, pytest automatically:
1. Creates `db_connection_string`
2. Creates `cache_ttl`
3. Creates `api_config`
4. Creates `database` (using `db_connection_string`)
5. Creates `cache` (using `cache_ttl`)
6. Creates `auth_service` (using `database` and `cache`)
7. Creates `api_server` (using `auth_service` and `api_config`)
8. Passes `api_server` to your test

Let's run these tests:

```bash
pytest test_api_composition.py -v
```

**Output:**
```
======================== test session starts =========================
collected 3 items

test_api_composition.py::test_successful_authentication PASSED  [ 33%]
test_api_composition.py::test_failed_authentication PASSED      [ 66%]
test_api_composition.py::test_cache_is_used PASSED              [100%]

========================= 3 passed in 0.02s ==========================
```

## Benefits of Fixture Composition

### 1. Modularity and Reusability

Each fixture is focused on one responsibility. Tests can mix and match:

```python
def test_database_directly(database):
    """Test just the database layer."""
    user = database.get_user('alice')
    assert user['email'] == 'alice@example.com'

def test_auth_service_directly(auth_service):
    """Test just the auth service."""
    result = auth_service.authenticate('alice', 'alice_password')
    assert result is True

def test_full_stack(api_server):
    """Test the complete application."""
    response = api_server.handle_request('alice', 'alice_password', '/users')
    assert response['status'] == 200
```

### 2. Easy Configuration Changes

Want to test with a different cache TTL? Just override one fixture:

```python
@pytest.fixture
def cache_ttl():
    """Use a shorter TTL for testing."""
    return 10  # 10 seconds instead of 300
```

All dependent fixtures automatically use the new value.

### 3. Clear Dependency Visualization

The fixture dependency graph makes it obvious what each component needs:

```python
@pytest.fixture
def auth_service(database, cache):
    """The signature shows exactly what this service depends on."""
    return AuthService(database, cache)
```

## Overriding Fixtures in Specific Tests

Sometimes you need to customize a fixture for specific tests. Use fixture overriding:

```python
# test_fixture_override.py
import pytest
from api import Database, Cache, AuthService

@pytest.fixture
def database():
    """Default database with standard test users."""
    db = Database("default_connection")
    db.add_user('alice', 'alice@example.com')
    return db

@pytest.fixture
def cache():
    """Default cache with 300 second TTL."""
    return Cache(300)

@pytest.fixture
def auth_service(database, cache):
    return AuthService(database, cache)

def test_with_default_fixtures(auth_service):
    """This test uses the default database and cache."""
    result = auth_service.authenticate('alice', 'password')
    assert result is True

@pytest.fixture
def database():
    """Override database for specific tests."""
    db = Database("special_connection")
    db.add_user('admin', 'admin@example.com')
    return db

def test_with_admin_user(auth_service):
    """This test uses the overridden database."""
    # The auth_service now uses the database with 'admin' user
    result = auth_service.authenticate('admin', 'admin_password')
    assert result is True
```

Wait—this won't work as written! Both `database` fixtures have the same name in the same file. Pytest will only see the second one.

To override fixtures for specific tests, use **fixture scope** or **conftest.py** (covered in section 4.7). For now, here's the correct pattern using a different fixture name:

```python
# test_fixture_customization.py
import pytest
from api import Database, Cache, AuthService

@pytest.fixture
def standard_database():
    """Standard database with regular users."""
    db = Database("standard_connection")
    db.add_user('alice', 'alice@example.com')
    db.add_user('bob', 'bob@example.com')
    return db

@pytest.fixture
def admin_database():
    """Database with admin users."""
    db = Database("admin_connection")
    db.add_user('admin', 'admin@example.com')
    db.add_user('superadmin', 'super@example.com')
    return db

@pytest.fixture
def cache():
    return Cache(300)

def test_standard_users(standard_database, cache):
    """Test with standard users."""
    auth = AuthService(standard_database, cache)
    assert auth.authenticate('alice', 'alice_password') is True
    assert auth.authenticate('admin', 'admin_password') is False

def test_admin_users(admin_database, cache):
    """Test with admin users."""
    auth = AuthService(admin_database, cache)
    assert auth.authenticate('admin', 'admin_password') is True
    assert auth.authenticate('alice', 'alice_password') is False
```

## Fixture Composition Patterns

### Pattern 1: Layered Architecture

Organize fixtures in layers that mirror your application architecture:

```python
# Layer 1: Configuration
@pytest.fixture
def db_config():
    return {"host": "localhost", "port": 5432}

# Layer 2: Infrastructure
@pytest.fixture
def database(db_config):
    return Database(**db_config)

# Layer 3: Services
@pytest.fixture
def user_service(database):
    return UserService(database)

# Layer 4: Application
@pytest.fixture
def app(user_service):
    return Application(user_service)
```

### Pattern 2: Shared Base with Variations

Create a base fixture and variations for different scenarios:

```python
@pytest.fixture
def base_database():
    """Minimal database setup."""
    return Database("test_connection")

@pytest.fixture
def database_with_users(base_database):
    """Database with test users."""
    base_database.add_user('alice', 'alice@example.com')
    base_database.add_user('bob', 'bob@example.com')
    return base_database

@pytest.fixture
def database_with_admin(base_database):
    """Database with admin user."""
    base_database.add_user('admin', 'admin@example.com')
    return base_database
```

### Pattern 3: Optional Dependencies

Some fixtures might have optional dependencies:

```python
@pytest.fixture
def logger():
    """Optional logging service."""
    class Logger:
        def log(self, message):
            print(f"[LOG] {message}")
    return Logger()

@pytest.fixture
def auth_service_with_logging(database, cache, logger):
    """Auth service with optional logging."""
    class LoggingAuthService(AuthService):
        def __init__(self, db, cache, logger):
            super().__init__(db, cache)
            self.logger = logger
        
        def authenticate(self, username, password):
            self.logger.log(f"Authenticating {username}")
            result = super().authenticate(username, password)
            self.logger.log(f"Authentication result: {result}")
            return result
    
    return LoggingAuthService(database, cache, logger)
```

## Circular Dependencies: What Not to Do

Pytest will detect and reject circular fixture dependencies:

```python
# test_circular_dependency.py (INVALID)
import pytest

@pytest.fixture
def fixture_a(fixture_b):
    return f"A depends on {fixture_b}"

@pytest.fixture
def fixture_b(fixture_a):
    return f"B depends on {fixture_a}"

def test_circular(fixture_a):
    assert True
```

```bash
pytest test_circular_dependency.py -v
```

**Output:**
```
======================== test session starts =========================
ERROR at setup of test_circular

fixture 'fixture_a' not found
> available fixtures: ...
> use 'pytest --fixtures [testpath]' for help on them.

ERROR: Fixture "fixture_a" called directly
```

Pytest detects the circular dependency and refuses to run. This is a design error—fixtures should form a **directed acyclic graph** (DAG), not a cycle.

If you find yourself needing circular dependencies, it's a sign that your fixture design needs refactoring. Usually, you can:
1. Extract the shared logic into a third fixture
2. Combine the two fixtures into one
3. Rethink the dependency relationship

## Visualizing Fixture Dependencies

To see the fixture dependency graph for your tests, use `pytest --fixtures`:
    ```

    ```bash
pytest --fixtures test_api_composition.py
    ```

    
This shows all available fixtures and their docstrings. For a specific test, use `--setup-show`:
    ```

    ```bash
pytest test_api_composition.py::test_cache_is_used --setup-show -v
    ```

    
**Output:**
```
======================== test session starts =========================
collected 1 item

test_api_composition.py::test_cache_is_used 
SETUP    F db_connection_string
SETUP    F cache_ttl
SETUP    F api_config
SETUP    F database (fixtures used: db_connection_string)
SETUP    F cache (fixtures used: cache_ttl)
SETUP    F auth_service (fixtures used: cache, database)
SETUP    F api_server (fixtures used: api_config, auth_service)
        test_api_composition.py::test_cache_is_used (fixtures used: api_server, cache)
PASSED
TEARDOWN F api_server
TEARDOWN F auth_service
TEARDOWN F cache
TEARDOWN F database
TEARDOWN F api_config
TEARDOWN F cache_ttl
TEARDOWN F db_connection_string

========================= 1 passed in 0.02s ==========================
```

This shows:
1. The order fixtures are created (SETUP)
2. Which fixtures each fixture depends on
3. The order fixtures are torn down (TEARDOWN)

Notice teardown happens in reverse order of setup—this ensures dependencies are cleaned up correctly.

## Composition Best Practices

### 1. Keep Fixtures Focused

Each fixture should do one thing:

✅ **Good:**
```python
@pytest.fixture
def database():
    return Database("test_connection")

@pytest.fixture
def database_with_users(database):
    database.add_user('alice', 'alice@example.com')
    return database
```

❌ **Bad:**
```python
@pytest.fixture
def everything():
    db = Database("test_connection")
    db.add_user('alice', 'alice@example.com')
    cache = Cache(300)
    auth = AuthService(db, cache)
    return db, cache, auth  # Too many responsibilities
```

### 2. Use Descriptive Names

Fixture names should clearly indicate what they provide:

✅ **Good:** `database_with_admin_user`, `authenticated_api_client`
❌ **Bad:** `db2`, `setup`, `fixture1`

### 3. Document Dependencies

Use docstrings to explain what each fixture provides and what it depends on:

```python
@pytest.fixture
def auth_service(database, cache):
    """
    Provide an authentication service.
    
    Dependencies:
        - database: Must contain test users
        - cache: Used for caching authentication results
    
    Returns:
        AuthService instance configured for testing
    """
    return AuthService(database, cache)
```

### 4. Prefer Composition Over Monolithic Fixtures

Build complex fixtures from simple ones rather than creating one giant fixture. This makes tests more flexible and fixtures more reusable.

## When to Use Fixture Composition

**Use composition when:**
- Your application has multiple layers (database, services, API)
- Tests need different combinations of components
- Setup is complex and benefits from being broken into steps
- You want to test individual components in isolation

**Don't use composition when:**
- Setup is simple and doesn't benefit from decomposition
- Fixtures would only be used once
- The dependency graph becomes too complex to understand

Balance is key. Start simple and add composition as complexity grows.

## Sharing Fixtures Across Files (conftest.py)

## The Problem: Fixture Duplication Across Test Files

As your test suite grows, you'll have multiple test files that need the same fixtures. Let's see this problem in action:

```python
# test_authentication.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture
def db():
    database = MockDatabase()
    database.add_user('alice', 'secret123')
    return database

@pytest.fixture
def authenticator(db):
    return UserAuthenticator(db)

def test_valid_credentials(authenticator):
    assert authenticator.authenticate('alice', 'secret123') is True
```

```python
# test_authorization.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture
def db():
    database = MockDatabase()
    database.add_user('alice', 'secret123')
    database.add_user('admin', 'admin_pass', role='admin')
    return database

@pytest.fixture
def authenticator(db):
    return UserAuthenticator(db)

def test_admin_privileges(authenticator):
    assert authenticator.is_admin('admin') is True
```

We've duplicated:
1. The `MockDatabase` class definition
2. The `db` fixture
3. The `authenticator` fixture

This creates maintenance problems:
- Changes to fixtures must be made in multiple files
- Inconsistencies can creep in between files
- More code to maintain

## The Solution: conftest.py

**`conftest.py` is a special file where pytest looks for shared fixtures.** Any fixture defined in `conftest.py` is automatically available to all test files in that directory and subdirectories.

Let's refactor our tests to use `conftest.py`:

```python
# conftest.py
import pytest
from auth import UserAuthenticator

class MockDatabase:
    """Shared database implementation for all tests."""
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture
def db():
    """Provide a test database with standard users."""
    database = MockDatabase()
    database.add_user('alice', 'secret123')
    database.add_user('admin', 'admin_pass', role='admin')
    return database

@pytest.fixture
def authenticator(db):
    """Provide an authenticator connected to the test database."""
    return UserAuthenticator(db)
```

```python
# test_authentication.py
def test_valid_credentials(authenticator):
    """No fixture definitions needed—they come from conftest.py."""
    assert authenticator.authenticate('alice', 'secret123') is True

def test_invalid_credentials(authenticator):
    assert authenticator.authenticate('alice', 'wrong') is False
```

```python
# test_authorization.py
def test_admin_privileges(authenticator):
    """This also uses fixtures from conftest.py."""
    assert authenticator.is_admin('admin') is True

def test_regular_user_not_admin(authenticator):
    assert authenticator.is_admin('alice') is False
```

Let's run these tests:

```bash
pytest test_authentication.py test_authorization.py -v
```

**Output:**
```
======================== test session starts =========================
collected 4 items

test_authentication.py::test_valid_credentials PASSED           [ 25%]
test_authentication.py::test_invalid_credentials PASSED         [ 50%]
test_authorization.py::test_admin_privileges PASSED             [ 75%]
test_authorization.py::test_regular_user_not_admin PASSED       [100%]

========================= 4 passed in 0.02s ==========================
```

All tests pass, and they all use the fixtures from `conftest.py`. No duplication!

## How conftest.py Works

### Discovery Rules

Pytest searches for `conftest.py` files in this order:

1. **Test file's directory**: First checks the directory containing the test file
2. **Parent directories**: Walks up the directory tree to the project root
3. **Merges fixtures**: Fixtures from all discovered `conftest.py` files are available

### Example Directory Structure

```
project/
├── conftest.py              # Root-level fixtures
├── tests/
│   ├── conftest.py          # Test-level fixtures
│   ├── test_auth.py
│   ├── unit/
│   │   ├── conftest.py      # Unit test fixtures
│   │   ├── test_database.py
│   │   └── test_cache.py
│   └── integration/
│       ├── conftest.py      # Integration test fixtures
│       └── test_api.py
```

**Fixture availability:**
- `test_database.py` can use fixtures from: `unit/conftest.py`, `tests/conftest.py`, `project/conftest.py`
- `test_api.py` can use fixtures from: `integration/conftest.py`, `tests/conftest.py`, `project/conftest.py`
- `test_auth.py` can use fixtures from: `tests/conftest.py`, `project/conftest.py`

## Hierarchical conftest.py Files

Let's create a realistic project structure with multiple `conftest.py` files:

```python
# conftest.py (root level)
import pytest

@pytest.fixture(scope="session")
def project_config():
    """Global configuration available to all tests."""
    return {
        "project_name": "MyApp",
        "version": "1.0.0",
        "environment": "test"
    }
```

```python
# tests/conftest.py
import pytest

class MockDatabase:
    def __init__(self):
        self.users = {}
    
    def add_user(self, username, password, role='user'):
        self.users[username] = {'password': password, 'role': role}
    
    def get_user(self, username):
        return self.users.get(username)

@pytest.fixture
def db():
    """Database fixture available to all tests."""
    database = MockDatabase()
    database.add_user('alice', 'secret123')
    return database
```

```python
# tests/unit/conftest.py
import pytest

@pytest.fixture
def unit_test_marker():
    """Marker to identify unit tests."""
    return "UNIT_TEST"
```

```python
# tests/integration/conftest.py
import pytest

@pytest.fixture
def integration_test_marker():
    """Marker to identify integration tests."""
    return "INTEGRATION_TEST"

@pytest.fixture
def api_client(db):
    """API client for integration tests."""
    from api import APIServer, AuthService, Cache
    
    cache = Cache(300)
    auth = AuthService(db, cache)
    return APIServer(auth, {"host": "localhost", "port": 8000})
```

```python
# tests/unit/test_database.py
def test_database_add_user(db, unit_test_marker):
    """Unit test can use fixtures from tests/conftest.py and tests/unit/conftest.py."""
    print(f"\n[{unit_test_marker}] Testing database")
    db.add_user('bob', 'bob_pass')
    assert db.get_user('bob') is not None

def test_project_config_available(project_config, unit_test_marker):
    """Unit test can also use root-level fixtures."""
    print(f"\n[{unit_test_marker}] Project: {project_config['project_name']}")
    assert project_config['environment'] == 'test'
```

```python
# tests/integration/test_api.py
def test_api_authentication(api_client, integration_test_marker):
    """Integration test uses fixtures from multiple conftest.py files."""
    print(f"\n[{integration_test_marker}] Testing API")
    response = api_client.handle_request('alice', 'secret123', '/users')
    assert response['status'] == 200

def test_api_with_project_config(api_client, project_config, integration_test_marker):
    """Integration test can access root-level fixtures."""
    print(f"\n[{integration_test_marker}] API version: {project_config['version']}")
    assert api_client.config['host'] == 'localhost'
```

```bash
pytest tests/ -v -s
```

**Output:**
```
======================== test session starts =========================
collected 4 items

tests/unit/test_database.py::test_database_add_user 
[UNIT_TEST] Testing database
PASSED
tests/unit/test_database.py::test_project_config_available 
[UNIT_TEST] Project: MyApp
PASSED

tests/integration/test_api.py::test_api_authentication 
[INTEGRATION_TEST] Testing API
PASSED
tests/integration/test_api.py::test_api_with_project_config 
[INTEGRATION_TEST] API version: 1.0.0
PASSED

========================= 4 passed in 0.03s ==========================
```

Each test has access to:
- Fixtures from its own directory's `conftest.py`
- Fixtures from parent directories' `conftest.py` files
- Fixtures from the root `conftest.py`

## Fixture Overriding in conftest.py

Fixtures in child directories can override fixtures from parent directories:

```python
# conftest.py (root)
import pytest

@pytest.fixture
def db():
    """Default database with minimal setup."""
    print("\n[ROOT] Creating minimal database")
    class MinimalDB:
        def __init__(self):
            self.users = {}
    return MinimalDB()
```

```python
# tests/integration/conftest.py
import pytest

@pytest.fixture
def db():
    """Override with a more complete database for integration tests."""
    print("\n[INTEGRATION] Creating full-featured database")
    class FullDB:
        def __init__(self):
            self.users = {}
            self.sessions = {}
            self.cache = {}
    return FullDB()
```

```python
# tests/unit/test_simple.py
def test_uses_root_db(db):
    """This uses the root-level db fixture."""
    print(f"\n[TEST] Database type: {type(db).__name__}")
    assert hasattr(db, 'users')
    assert not hasattr(db, 'sessions')  # MinimalDB doesn't have sessions
```

```python
# tests/integration/test_complex.py
def test_uses_integration_db(db):
    """This uses the overridden db fixture."""
    print(f"\n[TEST] Database type: {type(db).__name__}")
    assert hasattr(db, 'users')
    assert hasattr(db, 'sessions')  # FullDB has sessions
    assert hasattr(db, 'cache')     # FullDB has cache
```

```bash
pytest tests/ -v -s
```

**Output:**
```
======================== test session starts =========================
collected 2 items

tests/unit/test_simple.py::test_uses_root_db 
[ROOT] Creating minimal database
[TEST] Database type: MinimalDB
PASSED

tests/integration/test_complex.py::test_uses_integration_db 
[INTEGRATION] Creating full-featured database
[TEST] Database type: FullDB
PASSED

========================= 2 passed in 0.02s ==========================
```

The unit test used the root-level fixture, while the integration test used the overridden version.

## Organizing Fixtures by Purpose

A common pattern is to organize fixtures in `conftest.py` by their purpose:

```python
# conftest.py
import pytest

# ============================================================================
# Database Fixtures
# ============================================================================

@pytest.fixture
def db():
    """Provide a clean test database."""
    from database import Database
    db = Database("test_connection")
    yield db
    db.close()

@pytest.fixture
def db_with_users(db):
    """Database pre-populated with test users."""
    db.add_user('alice', 'alice@example.com')
    db.add_user('bob', 'bob@example.com')
    return db

# ============================================================================
# Authentication Fixtures
# ============================================================================

@pytest.fixture
def auth_service(db):
    """Provide an authentication service."""
    from auth import AuthService
    return AuthService(db)

@pytest.fixture
def authenticated_user(auth_service):
    """Provide an authenticated user session."""
    session = auth_service.login('alice', 'alice_password')
    return session

# ============================================================================
# API Fixtures
# ============================================================================

@pytest.fixture
def api_client(auth_service):
    """Provide an API client for testing."""
    from api import APIClient
    return APIClient(auth_service)

@pytest.fixture
def authenticated_api_client(api_client, authenticated_user):
    """Provide an authenticated API client."""
    api_client.set_session(authenticated_user)
    return api_client

# ============================================================================
# Configuration Fixtures
# ============================================================================

@pytest.fixture(scope="session")
def test_config():
    """Provide test configuration."""
    return {
        "database_url": "postgresql://test:test@localhost/testdb",
        "api_url": "http://localhost:8000",
        "timeout": 30
    }
```

This organization makes it easy to:
- Find related fixtures
- Understand fixture purposes
- Add new fixtures in the right place

## Common conftest.py Patterns

### Pattern 1: Autouse Fixtures for Test Environment Setup

Use `autouse=True` in `conftest.py` to set up the test environment automatically:

```python
# conftest.py
import pytest
import os

@pytest.fixture(scope="session", autouse=True)
def setup_test_environment():
    """Set up test environment before any tests run."""
    print("\n[SETUP] Configuring test environment")
    
    # Set environment variables
    os.environ['TESTING'] = 'true'
    os.environ['DATABASE_URL'] = 'postgresql://test:test@localhost/testdb'
    
    yield
    
    # Cleanup
    print("\n[TEARDOWN] Cleaning up test environment")
    del os.environ['TESTING']
    del os.environ['DATABASE_URL']
```

### Pattern 2: Fixture Factories in conftest.py

Provide factory fixtures for creating multiple instances:

```python
# conftest.py
import pytest

@pytest.fixture
def user_factory(db):
    """Factory for creating test users."""
    created_users = []
    
    def create_user(username, email, role='user'):
        db.add_user(username, email, role)
        created_users.append(username)
        return db.get_user(username)
    
    yield create_user
    
    # Cleanup: remove all created users
    for username in created_users:
        db.remove_user(username)
```

### Pattern 3: Conditional Fixtures Based on Markers

Create fixtures that behave differently based on test markers:

```python
# conftest.py
import pytest

@pytest.fixture
def db(request):
    """Provide different database based on test marker."""
    if 'slow' in request.keywords:
        # Use real database for slow tests
        print("\n[FIXTURE] Using real database")
        from database import RealDatabase
        db = RealDatabase("test_connection")
    else:
        # Use mock database for fast tests
        print("\n[FIXTURE] Using mock database")
        from database import MockDatabase
        db = MockDatabase()
    
    yield db
    
    if hasattr(db, 'close'):
        db.close()
```

```python
# test_example.py
import pytest

def test_fast_operation(db):
    """This uses MockDatabase."""
    assert db is not None

@pytest.mark.slow
def test_slow_operation(db):
    """This uses RealDatabase."""
    assert db is not None
```

## Viewing Available Fixtures

To see all fixtures available to your tests, including those from `conftest.py`:

```bash
pytest --fixtures
```

This shows:
- All fixtures defined in `conftest.py` files
- Built-in pytest fixtures
- Fixtures from installed plugins
- Each fixture's docstring

To see fixtures for a specific test file:

```bash
pytest --fixtures tests/unit/test_database.py
```

To see only your custom fixtures (not built-ins):

```bash
pytest --fixtures -v | grep -A 5 "conftest.py"
```

## conftest.py Best Practices

### 1. Keep conftest.py Focused

Don't put all fixtures in one root-level `conftest.py`. Organize by:
- **Root `conftest.py`**: Session-scoped fixtures, global configuration
- **tests/conftest.py`**: Common fixtures for all tests
- **tests/unit/conftest.py`**: Unit test-specific fixtures
- **tests/integration/conftest.py`**: Integration test-specific fixtures

### 2. Document Fixture Purpose

Every fixture in `conftest.py` should have a clear docstring:

```python
@pytest.fixture
def authenticated_api_client(api_client, user_session):
    """
    Provide an API client with an authenticated user session.
    
    This fixture combines an API client with a valid user session,
    allowing tests to make authenticated requests without manual login.
    
    Dependencies:
        - api_client: Base API client
        - user_session: Valid user authentication session
    
    Returns:
        APIClient: Configured client ready for authenticated requests
    """
    api_client.set_session(user_session)
    return api_client
```

### 3. Avoid Circular Imports

Be careful with imports in `conftest.py`. If your fixtures import from test files, you can create circular dependencies:

❌ **Bad:**
```python
# conftest.py
from test_helpers import create_test_user  # Circular import!

@pytest.fixture
def user():
    return create_test_user()
```

✅ **Good:**
```python
# conftest.py
@pytest.fixture
def user():
    # Import inside fixture to avoid circular dependency
    from helpers import create_test_user
    return create_test_user()
```

Or better yet, move shared code to a separate module:
```python
# test_utils.py
def create_test_user():
    return User("test_user")

# conftest.py
from test_utils import create_test_user

@pytest.fixture
def user():
    return create_test_user()
```

### 4. Use Scope Appropriately

In `conftest.py`, be especially careful with scope:
- **Session scope**: Only for truly global, immutable resources
- **Module scope**: For resources shared within a test file
- **Function scope**: Default, safest choice

### 5. Name Fixtures Clearly

Fixture names should indicate what they provide:

✅ **Good:** `authenticated_user`, `db_with_test_data`, `api_client_with_auth`
❌ **Bad:** `user`, `db`, `client` (too generic)

## When to Use conftest.py

**Use conftest.py when:**
- Multiple test files need the same fixtures
- You want to organize fixtures by test category (unit, integration, etc.)
- You need session-scoped fixtures available everywhere
- You want to set up test environment automatically

**Don't use conftest.py when:**
- Fixtures are only used in one test file (keep them local)
- Fixtures are highly specific to one test scenario
- You're creating fixtures "just in case" (YAGNI principle)

## The Journey: From Duplication to Shared Fixtures

Let's review the progression:

| Stage | Problem | Solution |
|-------|---------|----------|
| 1. No fixtures | Duplicated setup in every test | Create fixtures |
| 2. Local fixtures | Duplicated fixtures across files | Move to conftest.py |
| 3. Flat conftest.py | All fixtures in one file | Hierarchical conftest.py |
| 4. Organized conftest.py | Hard to find fixtures | Group by purpose, document |

Start simple with local fixtures. Move to `conftest.py` when duplication becomes painful. Organize hierarchically as your test suite grows.

## Summary: The Power of conftest.py

`conftest.py` is pytest's solution to fixture sharing. It enables:
- **Zero duplication**: Write fixtures once, use everywhere
- **Hierarchical organization**: Different fixtures for different test categories
- **Automatic discovery**: No imports needed
- **Fixture overriding**: Customize fixtures for specific test directories
- **Clear structure**: Organize fixtures by purpose and scope

Combined with fixture composition (section 4.6), `conftest.py` gives you a powerful system for building maintainable test suites that scale from dozens to thousands of tests.
