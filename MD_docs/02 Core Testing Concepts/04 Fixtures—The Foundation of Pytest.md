# Chapter 4: Fixtures—The Foundation of Pytest

## What Are Fixtures?

## What Are Fixtures?

In the world of testing, we often find ourselves writing the same setup and cleanup code over and over again. Imagine testing a user management system. For each test, you might need to:

1.  Connect to a database.
2.  Create a sample user.
3.  Perform the test action (e.g., update the user's email).
4.  Verify the result.
5.  Remove the sample user.
6.  Close the database connection.

Repeating steps 1, 2, 5, and 6 for every single test is tedious, error-prone, and makes your tests hard to read. This is the exact problem fixtures are designed to solve.

A **fixture** is a function that provides a reliable, repeatable context for your tests. It handles the setup before a test runs and the cleanup after it finishes. Think of it as a service that prepares the stage for your test and cleans up the props afterward.

### The Pain of Repetitive Setup

Let's look at a simple example without fixtures. We have a `Wallet` class and we want to test adding and spending cash.

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

Now, let's write some tests. Notice the repetition in creating a `Wallet` instance for each test.

```python
# tests/test_wallet_no_fixtures.py

from src.wallet import Wallet, InsufficientAmount
import pytest

def test_default_initial_amount():
    # Setup
    wallet = Wallet()
    # Assert
    assert wallet.balance == 0

def test_setting_initial_amount():
    # Setup
    wallet = Wallet(100)
    # Assert
    assert wallet.balance == 100

def test_wallet_add_cash():
    # Setup
    wallet = Wallet(20)
    # Action
    wallet.add_cash(80)
    # Assert
    assert wallet.balance == 100

def test_wallet_spend_cash():
    # Setup
    wallet = Wallet(20)
    # Action
    wallet.spend_cash(10)
    # Assert
    assert wallet.balance == 10

def test_wallet_spend_cash_raises_exception_on_insufficient_amount():
    # Setup
    wallet = Wallet()
    # Action & Assert
    with pytest.raises(InsufficientAmount):
        wallet.spend_cash(100)
```

This works, but the `wallet = Wallet(...)` line is repeated in every test. If the `Wallet` initialization became more complex (e.g., requiring a database connection or a configuration object), changing it in every test would be a nightmare.

### The Fixture Philosophy: Dependency Injection

Pytest fixtures introduce a powerful concept from software engineering called **dependency injection**. Instead of creating objects inside your test functions, your test functions *declare* what they need as arguments. Pytest sees these arguments, finds the corresponding fixtures, runs them, and "injects" the results into your test.

The test says, "I need a wallet to run." Pytest says, "I know how to make a wallet. Here you go."

This approach has profound benefits:
1.  **Decoupling**: Your test logic is separated from the setup logic. The test cares about *what* it's testing, not *how* the object is created.
2.  **Reusability**: The same fixture can be used by hundreds of tests across your entire project.
3.  **Maintainability**: If the `Wallet` initialization changes, you only need to update the fixture in one place.

Fixtures are the backbone of a clean, scalable, and maintainable pytest suite. They are more than just setup/teardown helpers; they are a powerful system for managing the resources and state your tests depend on.

## Simple Fixtures: Setup and Teardown

## Simple Fixtures: Setup and Teardown

The most basic fixture provides an object or value to a test. Let's refactor our wallet tests to use one. A fixture is just a Python function decorated with `@pytest.fixture`.

### A Basic Setup Fixture

We can create a fixture that provides an empty wallet. The name of the function (`wallet`) becomes the name of the fixture that tests can request.

```python
# tests/test_wallet_with_fixture.py

import pytest
from src.wallet import Wallet, InsufficientAmount

@pytest.fixture
def wallet():
    """Returns a Wallet instance with a balance of 20."""
    return Wallet(20)

def test_wallet_spend_cash(wallet):
    # The 'wallet' argument is automatically provided by pytest
    wallet.spend_cash(10)
    assert wallet.balance == 10

def test_wallet_add_cash(wallet):
    # This test also gets its own, separate Wallet instance
    wallet.add_cash(80)
    assert wallet.balance == 100
```

Look at how much cleaner the tests are! They declare their need for a `wallet`, and pytest provides it. Each test gets a *fresh* `Wallet` instance created by the fixture, ensuring that tests are isolated from each other.

### Adding Teardown with `yield`

What if our setup requires a corresponding cleanup action? For example, connecting to a database requires disconnecting later. Fixtures handle this elegantly using the `yield` keyword.

Code before the `yield` is the **setup** phase. The `yield` statement passes control to the test function, providing it with the resource. After the test function completes (whether it passes, fails, or raises an error), the code after the `yield` is executed as the **teardown** phase.

Let's create a simple example to see this in action. We'll use `print` statements to trace the execution flow.

```python
# tests/test_fixture_teardown.py
import pytest

@pytest.fixture
def setup_and_teardown():
    print("\n--- SETUP: Code before yield ---")
    data = {"key": "value"}
    yield data  # This is where the test runs
    print("\n--- TEARDOWN: Code after yield ---")
    data.clear()

def test_using_fixture(setup_and_teardown):
    print("\n>>> RUNNING TEST: test_using_fixture <<<")
    assert "key" in setup_and_teardown

def test_another_test(setup_and_teardown):
    print("\n>>> RUNNING TEST: test_another_test <<<")
    assert setup_and_teardown["key"] == "value"
```

Now, run this with the `-s` flag (to show print statements) and `-v` (for verbose output).

```bash
pytest -v -s tests/test_fixture_teardown.py
```

The output clearly shows the execution order for each test:

```text
=========================== test session starts ============================
...
collected 2 items

tests/test_fixture_teardown.py::test_using_fixture
--- SETUP: Code before yield ---

>>> RUNNING TEST: test_using_fixture <<<
PASSED
--- TEARDOWN: Code after yield ---

tests/test_fixture_teardown.py::test_another_test
--- SETUP: Code before yield ---

>>> RUNNING TEST: test_another_test <<<
PASSED
--- TEARDOWN: Code after yield ---

============================ 2 passed in ...s ==============================
```

Notice the pattern: **Setup -> Test -> Teardown**. This cycle repeats independently for each test that uses the fixture. This is the fundamental mechanism that guarantees test isolation.

## The @pytest.fixture Decorator

## The @pytest.fixture Decorator

The `@pytest.fixture` decorator is the entry point to this entire system. It signals to pytest that a function is not a test, but a factory for providing resources to tests.

Let's break down its role and common arguments.

### Basic Usage

As we've seen, the simplest form is just the decorator itself:

```python
@pytest.fixture
def my_data():
    return {"name": "Alice", "email": "alice@example.com"}
```

Any test function that includes `my_data` as a parameter will receive the dictionary returned by this fixture.

### Naming Fixtures

The name of the fixture is the name of the decorated function. It's a best practice to give fixtures descriptive names that clearly state what they provide, such as `db_connection`, `api_client`, or `sample_user_data`.

### Using `autouse` for Automatic Execution

Sometimes, you have a fixture that *every* test needs, but you don't want to add it as an argument to every single test function. A common example is a fixture that resets a database or clears a cache.

For this, you can use the `autouse=True` argument.

Let's create a fixture that prints a message before and after each test, without the tests needing to request it.

```python
# tests/test_autouse_fixture.py
import pytest

@pytest.fixture(autouse=True)
def announce_test_start_end():
    """A fixture that runs automatically for every test."""
    print("\n--- Test starting ---")
    yield
    print("\n--- Test ending ---")

def test_example_1():
    assert 1 == 1

def test_example_2():
    assert "a" in "abc"
```

Running this with `pytest -s` shows that the fixture runs for both tests, even though they don't explicitly request it:

```text
=========================== test session starts ============================
...
collected 2 items

tests/test_autouse_fixture.py::test_example_1
--- Test starting ---
PASSED
--- Test ending ---

tests/test_autouse_fixture.py::test_example_2
--- Test starting ---
PASSED
--- Test ending ---

============================ 2 passed in ...s ==============================
```

**A Word of Caution:** Use `autouse=True` sparingly. It can make it harder to understand where a test's setup is coming from, as the dependency is no longer explicitly declared in the test's signature. It's best reserved for broad, cross-cutting concerns like logging setup, database cleaning, or performance monitoring.

## Fixture Scope: Function, Class, Module, and Session

## Fixture Scope: Function, Class, Module, and Session

By default, a fixture is set up and torn down for **every single test function**. This is the `function` scope, and it provides the highest level of isolation. However, sometimes this is inefficient. If you need to connect to a database, you don't want to establish a new connection for each of the 500 tests in your suite.

Pytest allows you to control the lifecycle of a fixture using the `scope` argument. This is one of the most important concepts for writing efficient test suites.

The available scopes are, in order of increasing lifetime:
1.  `function`: The default. Runs once per test function.
2.  `class`: Runs once per test class.
3.  `module`: Runs once per test module (i.e., per `.py` file).
4.  `session`: Runs once per test session (i.e., once for the entire `pytest` command invocation).

### Visualizing Scopes

The best way to understand scopes is to see them in action. Let's create one fixture for each scope and use `print` statements to watch when they are created and destroyed.

```python
# tests/test_scopes.py
import pytest

@pytest.fixture(scope="session")
def session_fixture():
    print("\nSetting up SESSION fixture")
    yield
    print("\nTearing down SESSION fixture")

@pytest.fixture(scope="module")
def module_fixture():
    print("\nSetting up MODULE fixture")
    yield
    print("\nTearing down MODULE fixture")

@pytest.fixture(scope="class")
def class_fixture():
    print("\nSetting up CLASS fixture")
    yield
    print("\nTearing down CLASS fixture")

@pytest.fixture(scope="function")
def function_fixture():
    print("\nSetting up FUNCTION fixture")
    yield
    print("\nTearing down FUNCTION fixture")

def test_standalone(session_fixture, module_fixture, function_fixture):
    print(">> Running standalone test")

@pytest.mark.usefixtures("class_fixture")
class TestClass:
    def test_method_1(self, session_fixture, module_fixture, function_fixture):
        print(">> Running test_method_1")

    def test_method_2(self, session_fixture, module_fixture, function_fixture):
        print(">> Running test_method_2")
```

Now, run this file with `pytest -v -s`. The output is a perfect map of the fixture lifecycle.

```text
=========================== test session starts ============================
...
collected 3 items

tests/test_scopes.py::test_standalone
Setting up SESSION fixture
Setting up MODULE fixture
Setting up FUNCTION fixture
>> Running standalone test
PASSED
Tearing down FUNCTION fixture

tests/test_scopes.py::TestClass::test_method_1
Setting up CLASS fixture
Setting up FUNCTION fixture
>> Running test_method_1
PASSED
Tearing down FUNCTION fixture

tests/test_scopes.py::TestClass::test_method_2
Setting up FUNCTION fixture
>> Running test_method_2
PASSED
Tearing down FUNCTION fixture
Tearing down CLASS fixture
Tearing down MODULE fixture
Tearing down SESSION fixture

============================ 3 passed in ...s ==============================
```

Let's trace the execution:
1.  `session_fixture`: Sets up once at the very beginning. Tears down once at the very end.
2.  `module_fixture`: Sets up once before any test in `test_scopes.py` runs. Tears down after all tests in the file are complete.
3.  `class_fixture`: Sets up once before `test_method_1` (the first test in `TestClass`). Tears down after `test_method_2` (the last test in `TestClass`).
4.  `function_fixture`: Sets up and tears down around *each* of the three tests.

### Choosing the Right Scope

-   **`function` (Default):** Use for mutable objects that need to be reset for every test to ensure isolation (e.g., an empty list, a user object you plan to modify).
-   **`class`:** Useful when you have a group of tests in a class that all operate on the same expensive-to-create resource, and the tests don't modify the resource in a way that would affect other tests.
-   **`module`:** Ideal for resources that are read-only or can be shared across all tests in a file, like loading a large configuration or data file.
-   **`session`:** Use for very expensive, globally shared resources like a database connection pool or a web server instance that can be used by the entire test suite.

**Golden Rule:** Use the narrowest scope you can. Start with `function` and only increase the scope (`class`, `module`, `session`) when you have a clear performance reason to do so, and you are sure that sharing the fixture won't cause tests to interfere with each other.

## Using Fixtures in Your Tests

## Using Fixtures in Your Tests

There are two primary ways to apply a fixture to a test: by adding it as a function argument or by using a marker.

### Requesting Fixtures as Arguments

This is the most common and explicit method. By including the fixture's name as a parameter in your test function, you are telling pytest, "This test depends on this fixture." Pytest will then execute the fixture and pass its return value (or the value from `yield`) to the test.

```python
# tests/test_requesting_fixtures.py
import pytest

@pytest.fixture
def user_data():
    """Provides a dictionary of user data."""
    return {"name": "John Doe", "email": "john.doe@example.com"}

def test_user_name(user_data):
    # The `user_data` argument is filled by the fixture's return value
    assert user_data["name"] == "John Doe"

def test_user_email(user_data):
    assert user_data["email"] == "john.doe@example.com"
```

This method is preferred because it makes the test's dependencies crystal clear just by reading its signature.

### Using the `@pytest.mark.usefixtures` Marker

Sometimes, a fixture doesn't return a value; it just performs an action (like cleaning a database). In these cases, there's nothing to pass to the test function. For this, you can use the `@pytest.mark.usefixtures` marker.

This is useful for applying the same setup/teardown logic to an entire class of tests.

```python
# tests/test_usefixtures_marker.py
import pytest

@pytest.fixture
def clean_database():
    """A fixture that doesn't return anything, just performs actions."""
    print("\n... Cleaning database before test ...")
    yield
    print("\n... Cleaning database after test ...")

@pytest.mark.usefixtures("clean_database")
class TestUserOperations:
    def test_create_user(self):
        print(">> Running test_create_user")
        assert True

    def test_delete_user(self):
        print(">> Running test_delete_user")
        assert True
```

When you run this file with `pytest -s`, you'll see the "Cleaning database" messages appear before and after each test method in the class, even though `clean_database` is not listed as an argument. The marker applies the fixture to every test within its scope (in this case, the `TestUserOperations` class).

### Discovering Available Fixtures

How does pytest even know what fixtures are available? You can ask it directly! The `--fixtures` command-line flag will list all available fixtures, including built-in ones and those you've defined.

```bash
# Run this from your project's root directory
pytest --fixtures
```

The output will be a detailed list of every fixture, where it's defined, and its docstring. This is an incredibly useful tool for understanding the testing environment and for debugging issues where a fixture might not be found.

```text
=========================== test session starts ============================
...
======================= fixtures defined from ... ==========================
wallet
    tests/test_wallet_with_fixture.py:6
    Returns a Wallet instance with a balance of 20.

user_data
    tests/test_requesting_fixtures.py:4
    Provides a dictionary of user data.

... many built-in fixtures like capsys, tmp_path, etc. ...
```
This command banishes the "magic" and shows you exactly what pytest sees.

## Fixture Dependencies and Composition

## Fixture Dependencies and Composition

This is where the true power of pytest fixtures shines. **Fixtures can request other fixtures.** This allows you to build up complex test contexts from small, reusable, independent components. This is a classic example of the software design principle "composition over inheritance."

### The Problem: A "Fat" Fixture

Imagine setting up a test that requires a logged-in user who has placed an order. A naive approach might be to create one giant fixture that does everything.

```python
# (This is an anti-pattern example)
@pytest.fixture
def fat_fixture_for_order_test():
    # 1. Connect to DB
    db = connect_to_database()
    
    # 2. Create a user
    user = create_user(db, name="testuser")
    
    # 3. Log the user in
    api_client = APIClient()
    api_client.login(user)
    
    # 4. Create an order for the user
    order = create_order(db, user, items=["item1", "item2"])
    
    yield api_client, order
    
    # Teardown in reverse
    db.disconnect()
```

This is hard to read, hard to maintain, and impossible to reuse. What if you need just a database connection? Or just a logged-in client? You'd have to duplicate code.

### The Solution: Composable Fixtures

Let's refactor this into small, focused fixtures that depend on each other.

```python
# tests/test_composition.py
import pytest

# Fixture 1: The base resource (e.g., a database connection)
@pytest.fixture(scope="module")
def db_connection():
    print("\n(Connecting to database...)")
    connection = {"status": "connected"} # Fake connection object
    yield connection
    print("\n(Disconnecting from database...)")
    connection["status"] = "disconnected"

# Fixture 2: Depends on db_connection
@pytest.fixture
def api_client(db_connection):
    print(f"\n(Creating API client with {db_connection['status']} DB)")
    # This fixture can use the db_connection
    assert db_connection["status"] == "connected"
    client = {"user": None}
    yield client
    print("\n(Tearing down API client)")

# Fixture 3: Depends on api_client
@pytest.fixture
def logged_in_client(api_client):
    print("\n(Logging in client...)")
    api_client["user"] = "testuser"
    yield api_client
    print("\n(Logging out client...)")
    api_client["user"] = None

# The test requests the final, most complex fixture it needs.
def test_user_profile(logged_in_client):
    print("\n>> Running test_user_profile")
    assert logged_in_client["user"] == "testuser"

def test_guest_access(api_client):
    print("\n>> Running test_guest_access")
    assert api_client["user"] is None
```

Now, run this with `pytest -v -s`. Pay close attention to the setup order for `test_user_profile`.

```text
=========================== test session starts ============================
...
collected 2 items

tests/test_composition.py::test_user_profile
(Connecting to database...)
(Creating API client with connected DB)
(Logging in client...)

>> Running test_user_profile
PASSED
(Logging out client...)

(Tearing down API client)

tests/test_composition.py::test_guest_access
(Creating API client with connected DB)

>> Running test_guest_access
PASSED
(Tearing down API client)

(Disconnecting from database...)

============================ 2 passed in ...s ==============================
```

Notice the elegant dependency resolution:
1.  `test_user_profile` needs `logged_in_client`.
2.  `logged_in_client` needs `api_client`.
3.  `api_client` needs `db_connection`.
4.  Pytest executes them in the correct order: `db_connection` -> `api_client` -> `logged_in_client` -> test.
5.  The teardown happens in the exact reverse order.

For `test_guest_access`, which only needs `api_client`, pytest is smart enough to only run the `db_connection` and `api_client` fixtures.

This compositional pattern is the key to building a powerful and maintainable test suite. You create a library of building blocks (fixtures) and each test simply requests the final assembled product it needs.

## Sharing Fixtures Across Files (conftest.py)

## Sharing Fixtures Across Files (conftest.py)

As your test suite grows, you'll find that many fixtures are useful across multiple test files. For example, a `db_connection` fixture is likely needed by tests for users, products, and orders. Copying and pasting this fixture into every file would violate the DRY (Don't Repeat Yourself) principle.

Pytest solves this with a special file named `conftest.py`.

### The Role of `conftest.py`

When pytest runs, it searches for `conftest.py` files in the test directories. Any fixtures (and hooks, which we'll cover later) defined in a `conftest.py` file become automatically available to all tests in that directory and any of its subdirectories, without needing to be imported.

It acts as a local plugin or a shared utility module for your tests.

### Example: A Shared `data` Fixture

Let's set up a project structure like this:

```
project/
├── conftest.py
└── tests/
    ├── test_alpha.py
    └── subdir/
        └── test_beta.py
```

First, we define a shared fixture in the root `conftest.py`.

```python
# conftest.py

import pytest

@pytest.fixture(scope="session")
def shared_data():
    """A session-scoped fixture available to all tests."""
    print("\n(Setting up shared_data fixture)")
    return {"items": []}
```

Now, we can use this fixture in `test_alpha.py` without importing anything.

```python
# tests/test_alpha.py

def test_alpha_uses_shared_data(shared_data):
    """This test can access shared_data directly."""
    assert isinstance(shared_data["items"], list)
    shared_data["items"].append("alpha")
    print(f"\n>> test_alpha: shared_data is {shared_data}")
```

And we can also use it in a test file in a subdirectory.

```python
# tests/subdir/test_beta.py

def test_beta_uses_shared_data(shared_data):
    """This test, in a subdirectory, can also access it."""
    assert isinstance(shared_data["items"], list)
    shared_data["items"].append("beta")
    print(f"\n>> test_beta: shared_data is {shared_data}")
```

Finally, let's add one more test to see the final state of our session-scoped fixture.

```python
# tests/test_final_state.py

def test_final_state_of_shared_data(shared_data):
    """Checks the state after other tests have run."""
    print(f"\n>> test_final_state: shared_data is {shared_data}")
    assert "alpha" in shared_data["items"]
    assert "beta" in shared_data["items"]
```

Run `pytest -v -s`. The output will show that the same `shared_data` dictionary instance was passed to all tests.

```text
=========================== test session starts ============================
...
collected 3 items

(Setting up shared_data fixture)

tests/test_alpha.py::test_alpha_uses_shared_data
>> test_alpha: shared_data is {'items': ['alpha']}
PASSED

tests/subdir/test_beta.py::test_beta_uses_shared_data
>> test_beta: shared_data is {'items': ['alpha', 'beta']}
PASSED

tests/test_final_state.py::test_final_state_of_shared_data
>> test_final_state: shared_data is {'items': ['alpha', 'beta']}
PASSED

============================ 3 passed in ...s ==============================
```
This demonstrates both the power and the danger of higher-scoped fixtures. The `session` scope allowed us to share state, but it also means the tests are no longer perfectly isolated. `test_final_state` passes only because the other two tests ran first and modified the shared object. This is often undesirable, but `conftest.py` is the mechanism that enables it.

### Best Practices for `conftest.py`

-   Place your most general, widely-used fixtures (like database connections, API clients) in the root `conftest.py` of your `tests/` directory.
-   You can have multiple `conftest.py` files. A fixture in `tests/api/conftest.py` would be available to tests in `tests/api/` but not to tests in `tests/db/`. This allows you to scope your shared fixtures to specific parts of your application.
-   **Never** `import` anything from a `conftest.py` file. Pytest's discovery mechanism handles it for you.
