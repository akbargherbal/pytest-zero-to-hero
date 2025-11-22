# Chapter 20: Pro Tips, Best Practices, and Common Pitfalls

## Time-Saving Tips

## Time-Saving Tips

As you move from writing a few tests to managing large test suites, efficiency becomes paramount. The time spent waiting for tests to run is time you're not developing. This section covers powerful tools and techniques to tighten your feedback loop, run tests faster, and select only the tests you need, turning your test suite from a slow gatekeeper into a rapid development partner.

### 20.1.1 Watching Tests with pytest-watch

The core cycle of Test-Driven Development (TDD) is Red-Green-Refactor. You write a failing test (Red), write the code to make it pass (Green), and then clean up your code (Refactor). This cycle is most effective when it's fast. Manually re-running `pytest` after every small code change is tedious and breaks your flow.

The `pytest-watch` plugin automates this process. It monitors your project files for changes and automatically re-runs your tests whenever you save a file.

**Installation**

First, install the plugin.

```bash
pip install pytest-watch
```

**Usage**

Imagine you have a simple function and a test for it.

`src/utils.py`:

```python
# src/utils.py
def add(a, b):
    """A simple function to be tested."""
    return a + b
```

`tests/test_utils.py`:

```python
# tests/test_utils.py
from src.utils import add

def test_add_positive_numbers():
    assert add(2, 3) == 5

def test_add_negative_numbers():
    assert add(-2, -3) == -5
```

Instead of running `pytest` manually, start `pytest-watch` (often abbreviated as `ptw`) in your terminal:

```bash
ptw
```

It will run the tests once, then wait. Now, go into `src/utils.py` and introduce a bug. For example, change the return line to `return a - b`. As soon as you save the file, `pytest-watch` will detect the change and instantly re-run the tests, showing you the failures.

```text
...
=========================== short test summary info ===========================
FAILED tests/test_utils.py::test_add_positive_numbers - assert 2 + 3 == 5
FAILED tests/test_utils.py::test_add_negative_numbers - assert -2 + -3 == -5
...
>>> Waiting for changes...
```

Fix the bug, save the file, and the tests will run again, this time passing. This instant feedback loop is invaluable for maintaining focus and productivity. You can pass any standard pytest arguments to `ptw`, for example `ptw -k "positive"`.

## Parallel Test Execution with pytest-xdist

## Parallel Test Execution with pytest-xdist

As a project grows, its test suite can take minutes—or even hours—to run. `pytest-xdist` is a crucial plugin that dramatically speeds up test execution by running tests in parallel across multiple CPU cores.

**The Problem: Sequential Execution**

By default, pytest runs tests one by one. If you have 100 tests that each take 0.1 seconds, your total run time is 10 seconds. If you have 4 CPU cores, three of them are sitting idle.

**The Solution: Parallel Execution**

`pytest-xdist` distributes your tests across multiple worker processes, utilizing all available CPU power.

**Installation**

```bash
pip install pytest-xdist
```

**Usage**

The primary flag added by `pytest-xdist` is `-n` (or `--numprocesses`). You can specify a number of workers, or let it auto-detect the number of available CPU cores.

```bash
# Run tests in parallel using all available CPU cores
pytest -n auto

# Run tests using exactly 4 worker processes
pytest -n 4
```

Let's see it in action. Consider these tests, which simulate some I/O-bound work.

`tests/test_slow_operations.py`:

```python
# tests/test_slow_operations.py
import time
import pytest

@pytest.mark.parametrize("i", range(8))
def test_slow_operation(i):
    time.sleep(0.5)
    assert i >= 0
```

Running this normally would take at least 4 seconds (8 tests * 0.5s).

```bash
$ pytest --durations=3
# ...
# ========================= slowest 3 durations =========================
# 0.51s call     tests/test_slow_operations.py::test_slow_operation[1]
# 0.51s call     tests/test_slow_operations.py::test_slow_operation[0]
# 0.51s call     tests/test_slow_operations.py::test_slow_operation[2]
# ========================= 8 passed in 4.08s ===========================
```

Now, let's run it with `pytest-xdist` on a machine with 4 cores.

```bash
$ pytest -n auto --durations=0
# ...
# ========================= 8 passed in 1.15s ===========================
```

The total time is dramatically reduced. The 8 tests were distributed among the available workers, so instead of running 8 tasks sequentially, we ran 2 sets of 4 parallel tasks.

**Important Caveat: Test Interdependency**

`pytest-xdist` is a powerful tool, but it exposes a common anti-pattern: test interdependency. If `test_b` relies on some state created by `test_a`, your suite will fail unpredictably when run in parallel, because there's no guarantee `test_a` will run before `test_b` on the same worker. This forces you to write better, isolated tests, which is a best practice you should follow anyway (see Section 20.3.4).

## Test Selection Shortcuts

## Test Selection Shortcuts

Running the entire test suite is often unnecessary during development. You're typically working on a single feature or fixing a specific bug. Pytest's powerful test selection mechanisms, covered in Chapter 2, are your best friends for saving time. Here's a summary of the most useful shortcuts.

### Selecting by Keyword (`-k`)

The `-k` flag allows you to run tests whose names match a given expression.

Let's assume this test file:
`tests/test_user_auth.py`:

```python
# tests/test_user_auth.py
def test_user_can_login_with_valid_credentials():
    assert True

def test_user_cannot_login_with_invalid_password():
    assert True

def test_admin_can_access_dashboard():
    assert True

def test_user_cannot_access_admin_dashboard():
    assert True
```

Here are some examples of using `-k`:

```bash
# Run only tests related to login
pytest -k "login"

# Run tests for invalid scenarios
pytest -k "cannot"

# Run tests for admin users
pytest -k "admin"

# Use boolean operators for more complex queries
pytest -k "user and not admin"
```

### Selecting by Marker (`-m`)

Markers (Chapter 6) are tags you can apply to tests to categorize them. This is more robust than keyword matching.

```python
# tests/test_user_auth.py
import pytest

@pytest.mark.smoke
def test_user_can_login_with_valid_credentials():
    assert True

@pytest.mark.regression
def test_user_cannot_login_with_invalid_password():
    assert True

@pytest.mark.smoke
@pytest.mark.admin
def test_admin_can_access_dashboard():
    assert True

@pytest.mark.regression
def test_user_cannot_access_admin_dashboard():
    assert True
```

Now you can select tests by their category:

```bash
# Run only the quick "smoke" tests
pytest -m "smoke"

# Run only tests for the admin role
pytest -m "admin"

# Run smoke tests that are NOT admin tests
pytest -m "smoke and not admin"
```

### Selecting by File Path or Node ID

This is the most direct way to run a specific test or group of tests.

- **Run all tests in a directory**: `pytest tests/api/`
- **Run all tests in a file**: `pytest tests/test_user_auth.py`
- **Run a specific test function**: `pytest tests/test_user_auth.py::test_admin_can_access_dashboard`

Combining these shortcuts allows for surgical precision, saving you immense amounts of time by focusing the test runner only on the code you're actively changing.

## Using Test Templates

## Using Test Templates

Parametrization (Chapter 5) is the standard way to run the same test logic with different data. However, sometimes you have a set of tests that share a complex structure and behavior but aren't just simple data variations. In these cases, a "test template" pattern using class inheritance can be very effective.

The key is to define a base class with common tests but prevent pytest from discovering it as a test class directly. Then, you create concrete subclasses that provide the specific setup or data.

**The Problem: Repetitive Test Logic for Different Implementations**

Imagine you have two different database clients that are supposed to follow the same interface. You want to run the exact same suite of tests against both.

**The Solution: A Templated Base Class**

First, create a base class for your tests. We'll name it `BaseTestDatabaseClient` so pytest's default discovery (`Test*`) won't pick it up. This class will contain fixtures and test methods that rely on a `client` object, which is not yet defined.

```python
# tests/test_db_clients.py
import pytest

class BaseTestDatabaseClient:
    """
    A test template. Pytest will not collect this class because its name
    does not start with 'Test'.
    """
    @pytest.fixture
    def client(self):
        """This fixture must be overridden by subclasses."""
        raise NotImplementedError("Subclass must implement this fixture")

    def test_connect_disconnect(self, client):
        client.connect()
        assert client.is_connected()
        client.disconnect()
        assert not client.is_connected()

    def test_query_returns_data(self, client):
        client.connect()
        result = client.query("SELECT * FROM users")
        assert isinstance(result, list)
        client.disconnect()

# Dummy client implementations for the example
class PostgresClient:
    def __init__(self):
        self._connected = False
    def connect(self): self._connected = True
    def disconnect(self): self._connected = False
    def is_connected(self): return self._connected
    def query(self, q): return [{"id": 1, "name": "Alice"}]

class SQLiteClient:
    def __init__(self):
        self._connected = False
    def connect(self): self._connected = True
    def disconnect(self): self._connected = False
    def is_connected(self): return self._connected
    def query(self, q): return [{"id": 1, "name": "Bob"}]
```

Now, create concrete test classes that inherit from the base template. Each subclass only needs to provide the specific implementation of the `client` fixture.

```python
# tests/test_db_clients.py (continued)

class TestPostgresClient(BaseTestDatabaseClient):
    """
    Tests the PostgresClient against the standard interface.
    Pytest will collect this class.
    """
    @pytest.fixture
    def client(self):
        # Provide the concrete implementation for this test suite
        return PostgresClient()

class TestSQLiteClient(BaseTestDatabaseClient):
    """
    Tests the SQLiteClient against the standard interface.
    Pytest will collect this class.
    """
    @pytest.fixture
    def client(self):
        # Provide the concrete implementation for this test suite
        return SQLiteClient()
```

When you run `pytest`, it will discover `TestPostgresClient` and `TestSQLiteClient`. It will run `test_connect_disconnect` and `test_query_returns_data` for each of them, using the appropriate client implementation provided by the overridden fixture.

This pattern keeps your test logic DRY while allowing you to test multiple components that adhere to a common interface, a common scenario in plugin architectures or systems with multiple drivers.

## Industry Hacks and Patterns

## Industry Hacks and Patterns

Beyond the core features of pytest, several powerful patterns have emerged from real-world practice. These "hacks" address common, complex challenges like dealing with legacy code, managing enormous test suites, and ensuring distributed systems work together correctly.

### 20.2.1 Testing Legacy Code Without Refactoring

**The Problem:** You're tasked with modifying a complex piece of legacy code that has zero tests. You're afraid to change anything because you don't know what you might break. You can't refactor it to make it testable without tests to ensure your refactoring is safe. It's a classic catch-22.

**The Pattern: Characterization Tests**

A characterization test (also known as a "golden master" test) is not about verifying *correct* behavior. It's about verifying the *current* behavior, bugs and all. The goal is to create a safety net that locks down the system's existing output, allowing you to refactor with confidence.

**How to Write One**

1.  **Identify a pure function or a system boundary.** Find a piece of code that takes an input and produces a deterministic output (a value, a file, a log message).
2.  **Run the code with a representative input.**
3.  **Capture the output and save it.** This output is your "golden master." Store it in a file.
4.  **Write a test that runs the code again with the same input and asserts that the new output is identical to the saved golden master.**

**Example**

Imagine this convoluted legacy function you need to refactor:

```python
# src/legacy_report.py
def generate_report(data):
    # A complex, hard-to-read function with weird formatting and logic
    lines = []
    lines.append("--- REPORT ---")
    for i, item in enumerate(data):
        if item['value'] > 50:
            status = "HIGH"
        else:
            status = "low" # Inconsistent case
        lines.append(f"{i+1}. {item['name']}: {item['value']} [{status}]")
    lines.append("--- END ---")
    return "\n".join(lines)
```

You don't want to analyze its logic; you just want to preserve its behavior.

First, create a test file and a place to store the golden master.
`tests/test_legacy_report.py`:

```python
# tests/test_legacy_report.py
from pathlib import Path
from src.legacy_report import generate_report

# Define the path to our "golden master" file
GOLDEN_MASTER_PATH = Path(__file__).parent / "golden_master_report.txt"

def test_generate_report_characterization():
    sample_data = [
        {"name": "Sensor A", "value": 75},
        {"name": "Sensor B", "value": 30},
    ]

    # Generate the current output
    actual_output = generate_report(sample_data)

    # If the golden master doesn't exist, create it.
    # This is a one-time setup step.
    if not GOLDEN_MASTER_PATH.exists():
        GOLDEN_MASTER_PATH.write_text(actual_output)
        pytest.fail("Golden master file created. Re-run the test.")

    # Compare the current output to the golden master
    expected_output = GOLDEN_MASTER_PATH.read_text()
    assert actual_output == expected_output
```

The first time you run this test, it will fail, but it will create the `golden_master_report.txt` file:

`tests/golden_master_report.txt`:
```text
--- REPORT ---
1. Sensor A: 75 [HIGH]
2. Sensor B: 30 [low]
--- END ---
```

The second time you run the test, it will pass. Now, the behavior is "characterized." You can refactor `generate_report` with confidence. If you accidentally change the output (e.g., by fixing the capitalization of "low"), the test will fail, alerting you to the change. If the change was intentional, you simply delete the golden master file and re-run the test to generate a new one.

## Incremental Testing for Large Projects

## Incremental Testing for Large Projects

On massive projects, even with `pytest-xdist`, running the full test suite can be too slow for a tight feedback loop. Incremental testing strategies allow you to run only the tests most relevant to your recent changes.

### Run Only Failed Tests (`--last-failed` or `--lf`)

This is the simplest and most effective incremental strategy. After a test run, pytest saves a cache of which tests failed. The `--lf` flag tells pytest to run *only* the tests that failed on the last run.

**Workflow:**
1.  Run the full suite: `pytest`
2.  See 5 failures out of 2000 tests.
3.  Fix the code.
4.  Run `pytest --lf`. Pytest will now only execute those 5 tests.
5.  Once they pass, run `pytest --lf` again. Since there are no more known failures, pytest will run the full suite to ensure your fix didn't break anything else.

### Run Failed Tests First (`--failed-first` or `--ff`)

This is a variation of `--lf`. It runs the previously failed tests first, and if they all pass, it proceeds to run the rest of the tests. This gives you the fastest possible feedback on your fix while still providing the safety of a full run.

```bash
# Run last failed tests, then the rest if they pass
pytest --ff
```

### Testing Based on Code Changes

For even more advanced workflows, plugins can integrate with your version control system (like Git) to run only the tests that cover the code you've recently changed. A popular plugin for this is `pytest-testmon`. It monitors which tests execute which lines of code. When you change a file, it knows exactly which tests need to be re-run.

**Installation**

```bash
pip install pytest-testmon
```

**Usage**

```bash
# Run pytest with testmon enabled
pytest --testmon
```

The first run will be normal. After that, if you change a source file, `pytest --testmon` will only run the tests that were affected by that change, resulting in near-instantaneous feedback on large codebases.

## Contract Testing for Microservices

## Contract Testing for Microservices

**The Problem:** In a microservices architecture, you have a `UserService` and a `BillingService`. The `BillingService` depends on data from the `UserService`'s API. How do you ensure they can communicate without running slow, brittle end-to-end tests that require deploying both services? If the `UserService` team changes an API endpoint field from `user_id` to `userId`, the `BillingService` will break in production.

**The Pattern: Contract Testing**

Contract testing is a technique to ensure that two separate systems (e.g., an API provider and a consumer) are compatible without testing them directly together.

The consumer (`BillingService`) defines a "contract"—a file specifying the requests it will make and the responses it expects to receive. The provider (`UserService`) then uses this contract in its own test suite to verify that it fulfills the consumer's expectations.

While tools like Pact are industry standards for this, you can implement a simplified version of this pattern in pytest using a shared data file.

**Example Workflow**

1.  **The Consumer Defines the Contract**

    The `BillingService` team writes a test that mocks the `UserService` API. As part of this, they generate a contract file.

    `billing_service/tests/test_user_client.py`:

```python
# billing_service/tests/test_user_client.py
import json

def test_get_user_and_generate_contract():
    # This test defines what the BillingService expects from the UserService
    expected_user_data = {
        "userId": 123,
        "email": "test@example.com",
        "subscription_level": "premium"
    }

    # In a real test, you'd use this data to mock the API call
    # and test your client.
    # ... client logic test ...

    # Generate the contract file
    with open("../contracts/billing_service_expects_user.json", "w") as f:
        json.dump(expected_user_data, f, indent=2)

    assert True # Test passes by generating the contract
```

This contract is then committed to a shared repository or sent to the `UserService` team.

2.  **The Provider Verifies the Contract**

    The `UserService` team adds a test that uses this contract to validate their actual API endpoint.

    `user_service/tests/test_api_contracts.py`:

```python
# user_service/tests/test_api_contracts.py
import json
from user_service.api import get_user_by_id # The actual API implementation

def test_fulfills_billing_service_contract():
    # Load the contract defined by the consumer
    with open("../contracts/billing_service_expects_user.json") as f:
        contract = json.load(f)

    # Call the actual API endpoint
    user_id = contract["userId"]
    actual_user_data = get_user_by_id(user_id)

    # Verify that our actual response contains all the fields
    # the consumer expects.
    for key, value in contract.items():
        assert key in actual_user_data
        # You might want more specific type checks here too
        assert isinstance(actual_user_data[key], type(value))

    # This test now ensures we don't break the BillingService.
```

Now, if a developer on the `UserService` team renames `userId` to `user_id`, `test_fulfills_billing_service_contract` will fail in their CI pipeline, preventing them from deploying a breaking change long before it reaches production. This pattern allows teams to work independently while ensuring their services remain compatible.

## Property-Based Testing with Hypothesis

## Property-Based Testing with Hypothesis

So far, all our tests have been *example-based*. We, the developers, think of specific inputs (`add(2, 3)`) and assert a specific output (`== 5`). The weakness of this approach is that we might not think of the tricky edge cases.

*Property-based testing* flips this around. Instead of testing for specific outcomes, you state general properties of your code that should hold true for *all* valid inputs. A library then generates hundreds of different, often surprising, inputs to try and prove your property wrong.

The premier library for this in Python is `Hypothesis`.

**Installation**

```bash
pip install hypothesis
```

**Example: Testing an Encoding Function**

Let's say we have a function that encodes and decodes a string.

```python
# src/encoding.py
def encode(input_string: str) -> bytes:
    return input_string.encode("utf-8")

def decode(input_bytes: bytes) -> str:
    return input_bytes.decode("utf-8")
```

An example-based test might look like this:

```python
# tests/test_encoding_example.py
from src.encoding import encode, decode

def test_simple_string_roundtrip():
    original = "hello world"
    encoded = encode(original)
    decoded = decode(encoded)
    assert decoded == original
```

This is good, but what about empty strings? Strings with emoji? Strings with null bytes? We'd have to write a separate test for each case.

With Hypothesis, we describe the *property* we want to test: "for any valid text string, decoding the encoded version of it should result in the original string."

```python
# tests/test_encoding_property.py
from hypothesis import given
from hypothesis import strategies as st
from src.encoding import encode, decode

# The `given` decorator tells Hypothesis to run this test many times
# with different arguments.
# `st.text()` is a "strategy" that generates arbitrary text strings.
@given(st.text())
def test_string_roundtrip_property(original_string):
    """
    Property: Encoding and then decoding a string should return the original.
    """
    encoded = encode(original_string)
    decoded = decode(encoded)
    assert decoded == original_string
```

When you run this test, Hypothesis gets to work. It will generate simple strings (`""`, `"a"`), complex strings (`"你好"`, `"\x00"`), and long, convoluted strings. It intelligently searches for inputs that are likely to cause problems. If it finds an input that makes your assertion fail, it will report the *simplest possible failing example*.

For instance, if our `decode` function had a bug with non-ASCII characters, Hypothesis would run hundreds of examples and might report a failure like this:

```text
Falsifying example: test_string_roundtrip_property(original_string='€')
```

Property-based testing doesn't replace example-based testing—it complements it. Use examples for clear, simple business cases and properties for catching a wide range of edge cases in your algorithms and data processing logic.

## Best Practices Summary

## Best Practices Summary

Throughout this book, we've explored not just the "how" but also the "why" of testing. This section distills the most important philosophical principles and best practices into a concise summary. Following these guidelines will lead to a test suite that is not only effective but also readable, maintainable, and trustworthy.

### 20.3.1 Keep Tests Simple and Readable

A common debate in software is DRY (Don't Repeat Yourself) vs. DAMP (Descriptive and Meaningful Phrases). While DRY is a virtue in application code, in test code, DAMP is often more important.

A test should read like a story, explaining what is being set up, what action is being taken, and what outcome is expected. It's often better to repeat a few lines of setup code in two different tests than to hide them behind a complex fixture that makes the tests harder to understand in isolation.

**Bad: Overly DRY test**

```python
@pytest.fixture
def complex_user_setup(request):
    # ... 15 lines of complex setup logic based on request.param ...
    return user

@pytest.mark.parametrize("complex_user_setup", ["admin", "guest"], indirect=True)
def test_user_behavior(complex_user_setup):
    # What is this test actually verifying? It's hard to tell.
    assert complex_user_setup.can_do_thing()
```

**Good: Readable and explicit tests**

```python
def test_admin_user_can_do_thing():
    user = create_user(role="admin")
    # ... 2 lines of specific setup for this test ...
    assert user.can_do_thing()

def test_guest_user_cannot_do_thing():
    user = create_user(role="guest")
    # ... 2 lines of specific setup for this test ...
    assert not user.can_do_thing()
```

Clarity in tests is a feature. When a test fails six months from now, you should be able to understand what it's testing without having to debug the test itself.

### 20.3.2 One Assertion Per Test (Usually)

The ideal test focuses on a single concept or behavior. This makes test failures easier to diagnose. If a test with five assertions fails, you have to figure out which of the five behaviors is broken. If five separate tests fail, the test names themselves tell you exactly what's broken.

**Guideline:** A test should verify one logical concept.

This doesn't strictly mean one `assert` statement. It's perfectly fine to assert multiple properties of a single object if they form a coherent whole.

**Good: Multiple assertions for one concept**

```python
def test_create_user_returns_user_object_with_defaults():
    user = User.create(email="test@example.com")
    assert user.email == "test@example.com"
    assert user.is_active is True
    assert user.role == "viewer"
```

**Bad: Multiple concepts in one test**

```python
def test_user_creation_and_login():
    # Concept 1: User Creation
    user = User.create(email="test@example.com", password="pw")
    assert user.id is not None

    # Concept 2: User Login
    session_token = login(user.email, "pw")
    assert session_token is not None
```

If this test fails, is the problem with creation or login? Splitting it into `test_user_creation` and `test_user_login` makes the suite more precise.

### 20.3.3 Name Tests to Describe What They Test

Test names are documentation. A well-named test tells you what the code is supposed to do. When it fails, the name tells you what the code is *not* doing. A popular and effective naming convention is:

`test_unitOfWork_stateUnderTest_expectedBehavior`

**Bad:** `test_auth()`
**Slightly Better:** `test_login_fails()`
**Good:** `test_login_with_invalid_password_returns_error()`

Let's apply this to a real example:

```python
# Good, descriptive test names
def test_payment_processor_with_valid_card_succeeds():
    pass

def test_payment_processor_with_expired_card_raises_exception():
    pass

def test_payment_processor_when_fraud_service_is_down_uses_fallback():
    pass
```

When you see `FAILED: test_payment_processor_with_expired_card_raises_exception`, you know exactly what feature is broken without even looking at the code.

### 20.3.4 Avoid Test Interdependency

Each test should be a self-contained universe. It should be able to run independently and in any order relative to other tests. Relying on side effects from other tests is a recipe for a flaky and unreliable test suite.

**The Anti-Pattern:**

```python
# ANTI-PATTERN: DO NOT DO THIS
db = []

def test_add_item():
    db.append("item1")
    assert len(db) == 1

def test_remove_item():
    # This test depends on test_add_item running first!
    db.pop()
    assert len(db) == 0
```

If you run these tests with `pytest-xdist` or in a different order (`pytest tests.py::test_remove_item tests.py::test_add_item`), `test_remove_item` will fail.

**The Solution:** Use fixtures to guarantee a clean state for every test.

```python
# GOOD: Each test gets a clean state
import pytest

@pytest.fixture
def db():
    # The fixture provides a fresh database for each test
    return []

def test_add_item(db):
    db.append("item1")
    assert len(db) == 1

def test_list_is_initially_empty(db):
    # This test runs with its own empty `db` fixture instance
    assert len(db) == 0
```

This is one of the most critical principles for a scalable and reliable test suite.

### 20.3.5 Use Fixtures for Setup, Not Test Data

This is a subtle but important distinction.

-   **Fixtures** are for creating the *context* or *environment* your test runs in. Think database connections, authenticated user objects, temporary directories, or a running web server instance. They provide the "stage" for your test.
-   **`@pytest.mark.parametrize`** is for providing the *data* or *inputs* your test will act upon. Think different usernames, various numbers to be calculated, or different search queries.

**Blurring the lines (less ideal):**

```python
@pytest.fixture(params=["admin", "editor", "viewer"])
def user(request):
    return create_user(role=request.param)

def test_user_permissions(user):
    # This test is now doing three different things depending on the user role.
    # It's less clear what the specific intent is for each role.
    if user.role == "admin":
        assert user.can_delete()
    else:
        assert not user.can_delete()
```

**Clear separation (better):**

```python
@pytest.fixture
def admin_user():
    return create_user(role="admin")

@pytest.fixture
def viewer_user():
    return create_user(role="viewer")

def test_admin_can_delete(admin_user):
    assert admin_user.can_delete()

def test_viewer_cannot_delete(viewer_user):
    assert not viewer_user.can_delete()

# Use parametrize for variations on the same logical test
@pytest.mark.parametrize("username", ["valid_user", "user-with-dashes", "u"])
def test_valid_usernames(username):
    assert is_valid_username(username)
```

Using fixtures for context and parametrization for data leads to tests that are more explicit, readable, and easier to maintain.

## Common Pitfalls to Avoid

## Common Pitfalls to Avoid

Knowing what *not* to do is as important as knowing what to do. Many test suites become brittle, slow, and untrustworthy because of a few common anti-patterns. This section highlights these pitfalls and shows you how to steer clear of them.

### 20.4.1 Over-Mocking Your Code

Mocking (Chapter 8) is a powerful tool for isolating code, but it's easy to overuse. Over-mocking leads to "brittle" tests—tests that are so tightly coupled to the implementation details of your code that they break every time you refactor, even if the public behavior remains correct.

**The Pitfall:** A test that mocks every collaborator of the function under test.

```python
# src/service.py
from . import db, api, logger

def process_user_data(user_id):
    user = db.get_user(user_id)
    api_data = api.fetch_extra_data(user.email)
    user.update(api_data)
    db.save_user(user)
    logger.log(f"Processed user {user_id}")
    return "OK"

# tests/test_service.py
from unittest.mock import patch

# BAD: This test knows too much about the implementation
@patch("src.service.logger")
@patch("src.service.api")
@patch("src.service.db")
def test_process_user_data_brittle(mock_db, mock_api, mock_logger):
    # This test is just re-implementing the function logic in the test itself.
    # It doesn't actually test the integration of the components.
    mock_user = {"email": "test@example.com"}
    mock_db.get_user.return_value = mock_user
    mock_api.fetch_extra_data.return_value = {"extra": "data"}

    process_user_data(123)

    mock_db.get_user.assert_called_with(123)
    mock_api.fetch_extra_data.assert_called_with("test@example.com")
    mock_db.save_user.assert_called_with({"email": "test@example.com", "extra": "data"})
    mock_logger.log.assert_called_with("Processed user 123")
```

This test is fragile. If you decide to rename a variable or call the logger before saving the user, the test breaks, even though the end result is the same. The test is coupled to the *implementation*, not the *behavior*.

**The Solution:** Mock at the boundaries of your system. For an integration test like this, it's often better to use a real (but test-specific) database and only mock external services like the third-party API. This provides much more confidence that your components work together correctly.

### 20.4.2 Testing Implementation Instead of Behavior

This is a close cousin of over-mocking. A good test verifies the public contract or observable behavior of a unit of code. A bad test verifies the internal workings.

**The Pitfall:** Testing a private method.

```python
# src/calculator.py
class PriceCalculator:
    def __init__(self, tax_rate):
        self._tax_rate = tax_rate

    def _calculate_tax(self, price):
        return price * self._tax_rate

    def get_total_price(self, price):
        tax = self._calculate_tax(price)
        return price + tax

# tests/test_calculator.py
# BAD: Testing a private method directly
def test_calculate_tax_private_method():
    calc = PriceCalculator(tax_rate=0.1)
    # Python lets you do this, but you shouldn't.
    assert calc._calculate_tax(100) == 10
```

What's wrong with this? Imagine you refactor `PriceCalculator` to be more efficient, perhaps by inlining the calculation:

```python
# src/calculator.py (refactored)
class PriceCalculator:
    def __init__(self, tax_rate):
        self._tax_rate = tax_rate

    def get_total_price(self, price):
        # The private method is gone!
        return price * (1 + self._tax_rate)
```

The public behavior of `get_total_price` is identical, but `test_calculate_tax_private_method` now fails with an `AttributeError`. Your tests are hindering refactoring instead of enabling it.

**The Solution:** Test through the public interface.

```python
# GOOD: Testing the public behavior
def test_get_total_price_includes_tax():
    calc = PriceCalculator(tax_rate=0.1)
    assert calc.get_total_price(100) == 110
```

This test verifies the correct outcome regardless of how the tax is calculated internally. It will continue to pass after the refactoring, giving you confidence that you haven't broken anything.

### 20.4.3 Flaky Tests and Timing Issues

A flaky test is a test that sometimes passes and sometimes fails without any code changes. These are insidious because they destroy trust in your test suite. If developers see a test failing intermittently, they'll start to ignore it, and soon real failures will be missed.

**Common Causes and Solutions:**

1.  **Time Dependency:** Tests that use `datetime.now()` or `time.time()` can fail if they happen to run across a boundary (e.g., midnight).
    -   **Solution:** Use a library like `freezegun` to control the current time.

```python
# pip install freezegun
from freezegun import freeze_time

@freeze_time("2023-01-01 12:00:00")
def test_something_time_sensitive():
    # datetime.now() will always return the frozen time inside this test
    ...
```

2.  **Race Conditions:** In tests involving threading or asynchronous operations, you might assert a result before the operation has had time to complete.
    -   **Solution:** Use explicit synchronization (locks, events) or polling with a timeout. Avoid `time.sleep()` as it's unreliable and slow. For async code, properly `await` all operations.

3.  **Random Data:** Tests that rely on `random` can fail on an unlucky run.
    -   **Solution:** Set a fixed seed (`random.seed(0)`) at the beginning of the test to make the "random" sequence deterministic.

4.  **External Service Unavailability:** Tests that make real network calls to a third-party service will fail if that service is down or slow.
    -   **Solution:** Mock the external service (as discussed in Chapter 12). Your test suite should not depend on the internet.

### 20.4.4 Ignoring Test Maintenance

Tests are not write-only code. They are a living part of your codebase and require the same care and maintenance as your application code.

**The Pitfall:** A test suite full of commented-out tests, ignored failures (`@pytest.mark.xfail(reason="TODO: fix this")` that's been there for a year), and convoluted, unreadable test code. This is "test debt."

**The Solution:** Treat your tests as first-class citizens.
-   **Refactor tests:** When you refactor application code, refactor the corresponding tests to keep them clean and readable.
-   **Delete obsolete tests:** If you remove a feature, delete its tests. A large, slow test suite is a liability.
-   **Address failures immediately:** A failing test on the main branch should be treated as a critical bug. A clean test suite is a useful test suite.

### 20.4.5 Coverage Theater

As we saw in Chapter 13, code coverage is a useful metric, but it's not a measure of test quality. "Coverage theater" is the practice of chasing a 100% coverage number at the expense of writing meaningful tests.

**The Pitfall:** Writing tests that execute lines of code without actually asserting anything about their behavior.

```python
def process_data(data, mode="safe"):
    if not data:
        raise ValueError("Cannot process empty data")
    if mode == "fast":
        # ... complex logic ...
        return "fast_processed"
    else:
        # ... other complex logic ...
        return "safe_processed"

# BAD: This test achieves 100% coverage but is useless
def test_process_data_for_coverage():
    with pytest.raises(ValueError):
        process_data(None)
    process_data([1, 2], mode="fast")
    process_data([1, 2], mode="safe")
```

This test will give you 100% line coverage for `process_data`, but it asserts nothing about the return values. A bug could be introduced in the "fast" path, and this test would still pass.

**The Solution:** Focus on testing behaviors and properties. Write assertions that matter. It's better to have 80% coverage with strong assertions than 100% coverage with weak or missing assertions.

### 20.4.6 Tests That Pass When They Shouldn't

This is the most dangerous pitfall: a test that gives you a green checkmark but isn't actually testing what you think it is.

**The Pitfall:** A `pytest.raises` block that never raises.

```python
def test_divide_by_zero_should_raise_error():
    # Imagine we have a bug and `1 / 0` doesn't raise an error
    # or we accidentally test `1 / 1`.
    with pytest.raises(ZeroDivisionError):
        result = 1 / 1 # Oops, wrong input!

    # This test will PASS because no exception was raised *inside* the block,
    # and the code after the block is never reached. Pytest can't know
    # that the exception was *supposed* to happen.
```

The test passes silently, giving you a false sense of security.

**The Solution: The Red-Green-Refactor Cycle**

The best way to avoid this is to follow the TDD discipline: **always see the test fail first**.

1.  **Red:** Write the test for the feature that doesn't exist yet. Run it. It should fail (e.g., `NameError`, `AssertionError`). This proves your test is correctly wired up and capable of failing.
2.  **Green:** Write the minimum amount of application code to make the test pass.
3.  **Refactor:** Clean up both the application code and the test code.

If you write a test and it passes immediately, be suspicious. You may have written a test that can never fail.

## Where to Go From Here

## Where to Go From Here

Congratulations! You have journeyed from writing your first simple assertion to understanding the advanced patterns and philosophies that underpin professional software testing. You've built a solid foundation in pytest, but the journey of mastery is ongoing.

Here are some paths to continue your learning:

1.  **Explore the Plugin Ecosystem:** We've touched on essential plugins like `pytest-cov`, `pytest-xdist`, and `pytest-asyncio`. There are hundreds more. Browse the [official plugin list](https://docs.pytest.org/en/latest/reference/plugin_list.html) to find tools that can help with your specific domain, whether it's web development (e.g., `pytest-django`, `pytest-flask`), data science, or something else entirely.

2.  **Read the Official Documentation:** The official pytest documentation is an excellent, comprehensive resource. Now that you understand the core concepts, you'll be able to dive into the finer details of hooks, configuration, and internal mechanics.

3.  **Write Your Own Plugins:** The ultimate test of understanding is to extend the tool yourself. As we saw in Chapter 19, creating a simple plugin by implementing pytest hooks in `conftest.py` is surprisingly straightforward. Try writing a plugin that adds a custom command-line option or automatically adds a marker to certain tests.

4.  **Contribute to Open Source:** Find a project you use and look at their test suite. Can you improve it? Can you add tests for an un-tested part of the code? Reading and contributing to high-quality test suites written by experienced developers is one of the best ways to learn.

5.  **Teach Others:** The act of explaining a concept to someone else solidifies your own understanding. Mentor a junior developer, write a blog post about a clever testing trick you discovered, or give a presentation at a local meetup.

Testing is not a separate, secondary activity; it is an integral part of the craft of software development. The skills you've learned in this book will make you a more confident, effective, and professional developer. Go forth and build robust, reliable, and well-tested software.

## Cheat Sheet: Common Pytest Commands and Patterns

## Cheat Sheet: Common Pytest Commands and Patterns

A quick reference for the commands, markers, and patterns you'll use most frequently.

### Command-Line Flags
| Flag | Shorthand | Description |
| --- | --- | --- |
| `--verbose` | `-v` | Increase verbosity; show one test per line with status. |
| `--quiet` | `-q` | Decrease verbosity. |
| `--showlocals` | `-l` | Show local variables in tracebacks. |
| `--exitfirst` | `-x` | Stop the test session after the first failure. |
| `--maxfail=NUM` | | Stop after `NUM` failures. |
| `--keyword=EXPR` | `-k EXPR` | Run tests that match the given keyword expression. |
| `--mark=EXPR` | `-m EXPR` | Run tests that match the given marker expression. |
| `--collect-only` | | Show which tests would be run, without executing them. |
| `--pdb` | | Drop into the Python debugger on failure. |
| `--last-failed` | `--lf` | Run only the tests that failed in the last run. |
| `--failed-first` | `--ff` | Run last failed tests first, then the rest. |
| `--numprocesses=N`| `-n N` | (pytest-xdist) Run tests in parallel. Use `-n auto`. |
| `--cov=PATH` | | (pytest-cov) Generate a coverage report for the specified path. |
| `--durations=N` | | Show the `N` slowest tests. |

### Built-in Markers
| Marker | Description |
| --- | --- |
| `@pytest.mark.skip(reason=...)` | Always skip a test. |
| `@pytest.mark.skipif(condition, reason=...)` | Skip a test if the condition is true. |
| `@pytest.mark.xfail(reason=...)` | Expect a test to fail. It will be reported as `XFAIL` or `XPASS`. |
| `@pytest.mark.parametrize(argnames, argvalues)` | Perform parametrized testing. |
| `@pytest.mark.usefixtures("fixture_name")` | Explicitly use a fixture for a test, even if not an argument. |
| `@pytest.mark.filterwarnings("action:message")` | Add a warning filter for a specific test. |

### Core Assertions and Helpers
| Function | Description |
| --- | --- |
| `assert expression` | The primary way to make assertions. Pytest provides rich introspection. |
| `pytest.raises(ExpectedException)` | A context manager to assert that a block of code raises an exception. |
| `pytest.warns(ExpectedWarning)` | A context manager to assert that a block of code issues a warning. |
| `pytest.approx(expected, rel=..., abs=...)` | Assert that a floating-point number is approximately equal to another. |

### Fixture Scopes
| Scope | Description |
| --- | --- |
| `function` | (Default) The fixture is created once per test function. |
| `class` | The fixture is created once per test class. |
| `module` | The fixture is created once per module. |
| `session` | The fixture is created once for the entire test session. |

### Key `conftest.py` Hooks
| Hook | Description |
| --- | --- |
| `pytest_addoption(parser)` | Add custom command-line options. |
| `pytest_configure(config)` | Called after command-line options are parsed. Good for global setup. |
| `pytest_sessionstart(session)` | Called at the beginning of a test session. |
| `pytest_sessionfinish(session)` | Called at the end of a test session. |
| `pytest_generate_tests(metafunc)` | Allows for dynamic parametrization of tests. |

### Essential Plugins
| Plugin | Description |
| --- | --- |
| `pytest-cov` | Code coverage reporting. |
| `pytest-xdist` | Parallel test execution and other distribution features. |
| `pytest-watch` | Automatically re-run tests when files are modified. |
| `pytest-asyncio` | Support for testing `asyncio` code. |
| `pytest-benchmark` | Performance testing and benchmarking. |
| `hypothesis` | Advanced property-based testing. |
