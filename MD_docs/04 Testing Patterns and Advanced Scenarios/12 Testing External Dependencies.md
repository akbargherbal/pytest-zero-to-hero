# Chapter 12: Testing External Dependencies

## Database Testing Strategies

## The Challenge of the Outside World

So far, we have focused on testing "pure" functions—code whose output depends only on its input. This is the ideal scenario for testing. However, most real-world applications are not so simple. They interact with the outside world: they query databases, call external APIs, read from the filesystem, and send emails.

These interactions, known as **side effects** or **I/O (Input/Output)**, introduce significant challenges for testing:

*   **Slowness:** A network request or database query can take hundreds of milliseconds, orders of magnitude slower than a pure function call. A suite with hundreds of such tests can become unusably slow.
*   **Unreliability:** What if the network is down? The database is offline for maintenance? The API rate limit is exceeded? External systems can fail for reasons completely outside your control, causing your tests to fail sporadically. This is known as "flakiness."
*   **Statefulness:** A test that writes data to a database might affect the outcome of the next test that reads from it. This lack of **test isolation** makes tests order-dependent and hard to debug.
*   **Cost & Complexity:** Setting up a dedicated test database or paying for API calls during testing can be expensive and complicated.

The goal of this chapter is to learn strategies for taming these external dependencies, allowing us to write fast, reliable, and isolated tests for complex, real-world code.

### Phase 1: Establish the Reference Implementation

We will build our chapter around a single, realistic function that incorporates three common types of external dependencies: a database, an external API, and the filesystem.

Our anchor example is a function called `process_user_report`. Its job is to:
1.  Fetch a user's details from a **database**.
2.  Use the user's IP address to get their geographical location from an external **API**.
3.  Write a combined report to a **file**.

Let's start by defining the components of our system. We'll place this code in a new directory `user_reporter`.

```bash
mkdir user_reporter
touch user_reporter/__init__.py
touch user_reporter/main.py
touch user_reporter/db.py
touch user_reporter/api.py
```

First, the database interaction logic in `user_reporter/db.py`. We'll use SQLite for simplicity.

```python
# user_reporter/db.py
import sqlite3
from dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str
    ip_address: str

class UserDatabase:
    def __init__(self, db_path):
        self.db_path = db_path
        self.conn = sqlite3.connect(db_path)

    def get_user(self, user_id: int) -> User | None:
        cursor = self.conn.cursor()
        res = cursor.execute(
            "SELECT id, name, ip_address FROM users WHERE id = ?", (user_id,)
        ).fetchone()
        
        if res:
            return User(id=res[0], name=res[1], ip_address=res[2])
        return None

    def close(self):
        self.conn.close()

def setup_database(db_path):
    """A helper to create and populate the database for our example."""
    conn = sqlite3.connect(db_path)
    cursor = conn.cursor()
    cursor.execute("DROP TABLE IF EXISTS users")
    cursor.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            name TEXT NOT NULL,
            ip_address TEXT NOT NULL
        )
    """)
    cursor.execute(
        "INSERT INTO users (id, name, ip_address) VALUES (?, ?, ?)",
        (1, "Alice", "192.168.1.100")
    )
    conn.commit()
    conn.close()
```

Next, the API client in `user_reporter/api.py`. It calls a fictional geolocation API.

```python
# user_reporter/api.py
import requests

class GeolocationClient:
    def __init__(self, base_url="https://api.geolocation.com"):
        self.base_url = base_url

    def get_country_for_ip(self, ip_address: str) -> str:
        """Fetches the country for a given IP address."""
        response = requests.get(f"{self.base_url}/ip/{ip_address}")
        response.raise_for_status()  # Raise an exception for bad status codes
        return response.json()["country"]
```

Finally, the main logic that ties everything together in `user_reporter/main.py`.

```python
# user_reporter/main.py
from pathlib import Path
from .db import UserDatabase
from .api import GeolocationClient

def process_user_report(user_id: int, db_path: str, output_dir: Path):
    """
    Fetches user data, enriches it with geo IP info, and writes a report.
    """
    db = UserDatabase(db_path)
    user = db.get_user(user_id)
    db.close()

    if not user:
        raise ValueError(f"User with ID {user_id} not found.")

    api_client = GeolocationClient()
    country = api_client.get_country_for_ip(user.ip_address)

    report_path = output_dir / f"user_report_{user_id}.txt"
    report_content = f"User Report for {user.name}\n"
    report_content += f"IP Address: {user.ip_address}\n"
    report_content += f"Country: {country}\n"

    with open(report_path, "w") as f:
        f.write(report_content)
    
    print(f"Report generated at {report_path}")
```

### The Naive Integration Test

Now, let's write our first test for `process_user_report`. This test will be an *integration test*—it will interact with a real database file and attempt to make a real network call. This is the most straightforward, but also the most problematic, way to test our function.

We'll create a `tests` directory and our test file.

```bash
mkdir tests
touch tests/test_user_reporter.py
```

Here is our initial, problematic test.

```python
# tests/test_user_reporter.py
import os
from pathlib import Path
from user_reporter.main import process_user_report
from user_reporter.db import setup_database

# --- WARNING: This is a problematic test setup! ---

DB_PATH = "test_users.db"
OUTPUT_DIR = Path("./test_reports")

def test_process_user_report_naive():
    # 1. Setup: Create a real database file and output directory
    if os.path.exists(DB_PATH):
        os.remove(DB_PATH)
    setup_database(DB_PATH)
    OUTPUT_DIR.mkdir(exist_ok=True)

    # 2. Execute the function under test
    # This will make a REAL network request to a non-existent API!
    try:
        process_user_report(user_id=1, db_path=DB_PATH, output_dir=OUTPUT_DIR)
    except Exception as e:
        # We expect this to fail because the API doesn't exist.
        # For now, we'll just print the error to see what happens.
        print(f"API call failed as expected: {e}")
        # In a real scenario, this test would just fail unpredictably.

    # 3. Teardown: Clean up the created files
    if os.path.exists(DB_PATH):
        os.remove(DB_PATH)
    
    # We are not cleaning up the report file, which is another problem.
```

This test has numerous problems that we will solve one by one throughout this chapter:

1.  **Database Coupling:** It creates a real file (`test_users.db`) on disk. If two tests run in parallel, they could interfere with each other. Cleaning up this file manually is error-prone.
2.  **Network Dependency:** It tries to make a real HTTP request to `https://api.geolocation.com`. This will fail because the domain doesn't exist, but more importantly, it makes our test suite dependent on the network. It's slow and unreliable.
3.  **Filesystem Pollution:** It creates a `test_reports` directory and writes a file into it. It doesn't clean up this file, leaving artifacts behind after the test run.

In the following sections, we will systematically replace each of these real dependencies with test-friendly replacements, transforming this brittle integration test into a fast and reliable unit test.

## Using Fixtures for Database Setup

## Iteration 1: Isolating the Database

Our first problem is the hardcoded database file (`test_users.db`). Tests should never share state, and writing to a fixed file path is a classic example of shared state. If one test modifies the database, it could cause another test to fail.

### The Problem: Test Interference

Let's demonstrate this problem. Imagine we have two tests. The first test processes a user report, and a second test checks how many users are in the database.

```python
# tests/test_user_reporter_db_problem.py
import os
import sqlite3
from user_reporter.db import setup_database

DB_PATH = "shared_test.db"

def setup_module(module):
    """Set up the database once for all tests in this file."""
    if os.path.exists(DB_PATH):
        os.remove(DB_PATH)
    setup_database(DB_PATH)
    print(f"\n--- Database '{DB_PATH}' set up ---")

def teardown_module(module):
    """Tear down the database once after all tests."""
    if os.path.exists(DB_PATH):
        os.remove(DB_PATH)
    print(f"\n--- Database '{DB_PATH}' torn down ---")

def test_adds_a_new_user():
    """This test modifies the database state."""
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    cursor.execute(
        "INSERT INTO users (id, name, ip_address) VALUES (?, ?, ?)",
        (2, "Bob", "10.0.0.5")
    )
    conn.commit()
    conn.close()
    
    # Check that the user was added
    conn = sqlite3.connect(DB_PATH)
    count = conn.cursor().execute("SELECT COUNT(*) FROM users").fetchone()[0]
    conn.close()
    assert count == 2

def test_initial_user_count_is_one():
    """This test assumes a clean initial state."""
    conn = sqlite3.connect(DB_PATH)
    cursor = conn.cursor()
    count = cursor.execute("SELECT COUNT(*) FROM users").fetchone()[0]
    conn.close()
    assert count == 1
```

Let's run this file with `pytest -v -s`. The `-s` flag is important to see our `print` statements. Pytest will run tests in alphabetical order, so `test_adds_a_new_user` runs first.

```bash
$ pytest -v -s tests/test_user_reporter_db_problem.py
=========================== test session starts ============================
...
collected 2 items

--- Database 'shared_test.db' set up ---
tests/test_user_reporter_db_problem.py::test_adds_a_new_user PASSED
tests/test_user_reporter_db_problem.py::test_initial_user_count_is_one FAILED

================================= FAILURES =================================
____________ test_initial_user_count_is_one ____________

    def test_initial_user_count_is_one():
        """This test assumes a clean initial state."""
        conn = sqlite3.connect(DB_PATH)
        cursor = conn.cursor()
        count = cursor.execute("SELECT COUNT(*) FROM users").fetchone()[0]
        conn.close()
>       assert count == 1
E       assert 2 == 1

tests/test_user_reporter_db_problem.py:40: AssertionError
--- Database 'shared_test.db' torn down ---
========================= 1 failed, 1 passed in ...s =========================
```

### Diagnostic Analysis: Reading the Failure

**The complete output**: The output clearly shows one test passing and one failing with an `AssertionError`.

**Let's parse this section by section**:

1.  **The summary line**: `FAILED tests/test_user_reporter_db_problem.py::test_initial_user_count_is_one - AssertionError`
    *   What this tells us: The test named `test_initial_user_count_is_one` failed because an `assert` statement was false.

2.  **The traceback**: The traceback is very short, pointing directly to the failing line: `assert count == 1`.

3.  **The assertion introspection**:
    ```
    E       assert 2 == 1
    ```
    *   What this tells us: Pytest's excellent introspection shows us the values at the time of the assertion. The variable `count` was `2`, but the test expected it to be `1`.

**Root cause identified**: The `test_adds_a_new_user` test ran first and added a user to the database. The `test_initial_user_count_is_one` test ran second and saw the modified database, not the clean state it expected. The tests are not isolated.

**Why the current approach can't solve this**: Using `setup_module` and `teardown_module` creates a shared database for all tests in the file. This is efficient but leads to state pollution.

**What we need**: A mechanism to provide each test function with its own, clean, isolated database session, and automatically clean up any changes after the test finishes. This is a perfect job for fixtures.

### The Solution: Transactional Database Fixtures

We will create a fixture that provides a database session. For each test, it will:
1.  Begin a transaction.
2.  Yield the database session to the test.
3.  After the test completes, roll back the transaction.

Rolling back the transaction effectively erases any changes the test made to the database, ensuring the next test starts from a pristine state. We'll use an in-memory SQLite database to avoid creating files altogether.

Let's create a `conftest.py` file to hold our new fixture.

```python
# tests/conftest.py
import pytest
import sqlite3
from user_reporter.db import User

@pytest.fixture(scope="session")
def db_connection():
    """A session-scoped fixture for a single in-memory DB connection."""
    conn = sqlite3.connect(":memory:")
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            name TEXT NOT NULL,
            ip_address TEXT NOT NULL
        )
    """)
    conn.commit()
    yield conn
    conn.close()

@pytest.fixture
def db_session(db_connection):
    """
    A function-scoped fixture to provide a clean database state for each test.
    """
    # Seed the database with initial data for each test
    cursor = db_connection.cursor()
    cursor.execute(
        "INSERT INTO users (id, name, ip_address) VALUES (?, ?, ?)",
        (1, "Alice", "192.168.1.100")
    )
    db_connection.commit()

    yield db_connection

    # Teardown: Clean up by deleting all data
    cursor.execute("DELETE FROM users")
    db_connection.commit()
```

Let's break this down:
*   `db_connection`: This is a `session`-scoped fixture. It creates a single in-memory database connection that lasts for the entire test run. This is efficient. It also creates the `users` table just once.
*   `db_session`: This is a `function`-scoped fixture (the default scope). It runs for *every single test function* that requests it.
    *   It depends on `db_connection` to get the shared connection.
    *   Before yielding to the test, it inserts our initial "Alice" user. This ensures every test starts with the exact same known data.
    *   After the test finishes, the `yield` statement resumes, and it runs the teardown code: `DELETE FROM users`. This wipes the table clean, guaranteeing isolation for the next test.

Now, let's rewrite our failing tests to use this fixture.

```python
# tests/test_user_reporter_db_solution.py
import sqlite3

def test_adds_a_new_user_isolated(db_session):
    """This test modifies the database state, but changes are isolated."""
    cursor = db_session.cursor()
    cursor.execute(
        "INSERT INTO users (id, name, ip_address) VALUES (?, ?, ?)",
        (2, "Bob", "10.0.0.5")
    )
    db_session.commit()
    
    count = cursor.execute("SELECT COUNT(*) FROM users").fetchone()[0]
    assert count == 2

def test_initial_user_count_is_one_isolated(db_session):
    """This test now gets a clean state and passes."""
    cursor = db_session.cursor()
    count = cursor.execute("SELECT COUNT(*) FROM users").fetchone()[0]
    assert count == 1
```

### Verification

Let's run the new test file.

```bash
$ pytest -v tests/test_user_reporter_db_solution.py
=========================== test session starts ============================
...
collected 2 items

tests/test_user_reporter_db_solution.py::test_adds_a_new_user_isolated PASSED [ 50%]
tests/test_user_reporter_db_solution.py::test_initial_user_count_is_one_isolated PASSED [100%]

============================ 2 passed in ...s ==============================
```

Success! Both tests pass, regardless of the order they are run in. The `db_session` fixture ensures that `test_initial_user_count_is_one_isolated` sees only the single "Alice" user that was set up for it, because the "Bob" user added by the other test was cleaned up automatically.

We have successfully isolated our tests from the database state. Now, let's apply this to our main `process_user_report` function. But first, we need to make a small but critical change to our application code to make it more testable. This pattern is called **Dependency Injection**.

## Testing API Calls

## Iteration 2: Taming the Network

Our tests are now isolated from the database, but our main function `process_user_report` still has a major problem: it instantiates its own `GeolocationClient` and makes a real network call.

```python
# user_reporter/main.py (Original)
def process_user_report(...):
    # ...
    api_client = GeolocationClient() # Problem: Hardcoded dependency
    country = api_client.get_country_for_ip(user.ip_address)
    # ...
```

This makes testing difficult. We can't easily replace the real `GeolocationClient` with a fake one for testing. The solution is to refactor the function to accept its dependencies as arguments.

### Refactoring for Testability: Dependency Injection

Let's modify `process_user_report` and the other components to allow dependencies to be "injected".

First, we'll update `UserDatabase` to accept a connection object instead of creating its own. This allows our fixture to provide the connection.

```python
# user_reporter/db.py (Updated)
import sqlite3
from dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str
    ip_address: str

class UserDatabase:
    # It now accepts an existing connection
    def __init__(self, connection: sqlite3.Connection):
        self.conn = connection

    def get_user(self, user_id: int) -> User | None:
        # ... (rest of the method is the same)
        cursor = self.conn.cursor()
        res = cursor.execute(
            "SELECT id, name, ip_address FROM users WHERE id = ?", (user_id,)
        ).fetchone()
        
        if res:
            return User(id=res[0], name=res[1], ip_address=res[2])
        return None

    # No longer needs a close() method, as the connection is managed externally
```

Now, the main function. It will now accept instances of `UserDatabase` and `GeolocationClient`.

```python
# user_reporter/main.py (Updated)
from pathlib import Path
from .db import UserDatabase
from .api import GeolocationClient

def process_user_report(
    user_id: int, 
    db: UserDatabase, 
    api_client: GeolocationClient, 
    output_dir: Path
):
    """
    Fetches user data, enriches it with geo IP info, and writes a report.
    Dependencies are now injected.
    """
    user = db.get_user(user_id)

    if not user:
        raise ValueError(f"User with ID {user_id} not found.")

    country = api_client.get_country_for_ip(user.ip_address)

    report_path = output_dir / f"user_report_{user_id}.txt"
    report_content = f"User Report for {user.name}\n"
    report_content += f"IP Address: {user.ip_address}\n"
    report_content += f"Country: {country}\n"

    with open(report_path, "w") as f:
        f.write(report_content)
    
    print(f"Report generated at {report_path}")
```

This refactoring is a game-changer. Our function is no longer responsible for creating its dependencies; it just uses the ones it's given. In production, we'll pass in real objects. In testing, we can pass in fakes.

### The Problem: Unreliable Network Calls

Let's write a test for our refactored function. We'll use our `db_session` fixture, but for now, we'll still use a real `GeolocationClient`.

```python
# tests/test_user_reporter.py
from pathlib import Path
from user_reporter.main import process_user_report
from user_reporter.db import UserDatabase
from user_reporter.api import GeolocationClient

def test_process_user_report_network_problem(db_session):
    # Arrange: Use our DB fixture and a real API client
    db = UserDatabase(db_session)
    api_client = GeolocationClient(base_url="https://api.geolocation.com") # Non-existent API
    output_dir = Path("./reports") # Another problem: writing to a real directory
    output_dir.mkdir(exist_ok=True)

    # Act & Assert: This will fail because of the network call
    process_user_report(
        user_id=1,
        db=db,
        api_client=api_client,
        output_dir=output_dir
    )
```

Let's run this test. It will fail, but the failure is informative.

```bash
$ pytest -v tests/test_user_reporter.py::test_process_user_report_network_problem
=========================== test session starts ============================
...
collected 1 item

tests/test_user_reporter.py::test_process_user_report_network_problem FAILED [100%]

================================= FAILURES =================================
_______ test_process_user_report_network_problem _______

db_session = <sqlite3.Connection object at 0x...>

    def test_process_user_report_network_problem(db_session):
        # ... (Arrange)
    
        # Act & Assert: This will fail because of the network call
>       process_user_report(
            user_id=1,
            db=db,
            api_client=api_client,
            output_dir=output_dir
        )

tests/test_user_reporter.py:16: 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
user_reporter/main.py:17: in process_user_report
    country = api_client.get_country_for_ip(user.ip_address)
user_reporter/api.py:9: in get_country_for_ip
    response = requests.get(f"{self.base_url}/ip/{ip_address}")
...
    raise ConnectionError(e, request=request)
E   requests.exceptions.ConnectionError: HTTPSConnectionPool(host='api.geolocation.com', port=443): Max retries exceeded with url: /ip/192.168.1.100 (Caused by NameResolutionError("<urllib3.connection.HTTPSConnection object at 0x...>: Failed to resolve 'api.geolocation.com' ([Errno 8] nodename nor servname provided, or not known)"))
========================= 1 failed in ...s =========================
```

### Diagnostic Analysis: Reading the Failure

**The complete output**: The test fails with a `requests.exceptions.ConnectionError`.

**Let's parse this section by section**:

1.  **The summary line**: `FAILED ... - requests.exceptions.ConnectionError`
    *   What this tells us: The test failed because the `requests` library couldn't connect to the server.

2.  **The traceback**:
    ```
    user_reporter/main.py:17: in process_user_report
        country = api_client.get_country_for_ip(user.ip_address)
    user_reporter/api.py:9: in get_country_for_ip
        response = requests.get(f"{self.base_url}/ip/{ip_address}")
    ```
    *   What this tells us: The failure originated deep inside the `requests` library, but the call chain shows it was triggered by `api_client.get_country_for_ip` inside our `process_user_report` function.

3.  **The exception details**: `Failed to resolve 'api.geolocation.com'`
    *   What this tells us: The specific reason for the connection error is that the domain name could not be found. This is expected, as we made it up. If it were a real API that was temporarily down, we might see a timeout error or a different connection error.

**Root cause identified**: The test is attempting to make a real HTTP request to an external service, making it slow and subject to network failures.

**Why the current approach can't solve this**: We are passing a real `GeolocationClient` to our function. Its entire purpose is to make real network calls.

**What we need**: A way to intercept the outgoing HTTP request made by the `requests` library and return a fake, predefined response without ever touching the network. This is called **mocking**.

## Mocking HTTP Requests with responses

## The Solution: Mocking with the `responses` Library

While `unittest.mock` (covered in Chapter 11) can be used to patch the `requests` library, there are specialized tools that make mocking HTTP requests much cleaner and more powerful. One of the best is the `responses` library.

First, let's install it.

```bash
pip install responses
```

The `responses` library works as a context manager or a decorator that patches `requests` for you. Inside its context, any call to `requests.get`, `requests.post`, etc., is intercepted. If the call matches a URL you've registered, your predefined fake response is returned. If it doesn't match, it raises an error.

### Creating a Mocking Fixture

We can wrap the `responses` functionality in a pytest fixture for easy reuse. Let's add this to our `tests/conftest.py`.

```python
# tests/conftest.py (additions)
import responses

@pytest.fixture
def mock_responses():
    """A fixture to mock out the requests library."""
    with responses.RequestsMock() as rsps:
        yield rsps
```

This fixture starts a `RequestsMock` context, yields the mock object (`rsps`) to the test so it can register fake URLs, and ensures everything is cleaned up afterward.

### Iteration 2 Solution: Applying the Mock

Now we can rewrite our failing test to use this fixture. We will inject a real `GeolocationClient`, but because our `mock_responses` fixture is active, the client's call to `requests.get` will be intercepted and never reach the network.

```python
# tests/test_user_reporter.py (updated test)
from pathlib import Path
import responses # Import the library
from user_reporter.main import process_user_report
from user_reporter.db import UserDatabase
from user_reporter.api import GeolocationClient

# We still have the filesystem problem, which we'll fix next.
# For now, we'll just use a dummy path and not check the file contents.
DUMMY_OUTPUT_DIR = Path("./dummy_reports")

def test_process_user_report_with_mock_api(db_session, mock_responses):
    # Arrange (Database)
    db = UserDatabase(db_session)

    # Arrange (API Mock)
    # We tell `responses` that any GET request to this specific URL...
    mock_responses.get(
        "https://api.geolocation.com/ip/192.168.1.100",
        json={"country": "USA"}, # ...should return this JSON payload...
        status=200, # ...with a 200 OK status code.
    )
    # We still pass a real API client, but its requests will be intercepted.
    api_client = GeolocationClient(base_url="https://api.geolocation.com")
    
    # Arrange (Filesystem)
    DUMMY_OUTPUT_DIR.mkdir(exist_ok=True)

    # Act
    process_user_report(
        user_id=1,
        db=db,
        api_client=api_client,
        output_dir=DUMMY_OUTPUT_DIR
    )

    # Assert
    # For now, we can't easily check the file. We'll just assert the test runs without error.
    # We will add a proper file assertion in the next section.
    assert len(mock_responses.calls) == 1
    assert mock_responses.calls[0].request.url == "https://api.geolocation.com/ip/192.168.1.100"
    assert mock_responses.calls[0].response.status_code == 200
```

### Verification

Let's run our new test.

```bash
$ pytest -v tests/test_user_reporter.py::test_process_user_report_with_mock_api
=========================== test session starts ============================
...
collected 1 item

tests/test_user_reporter.py::test_process_user_report_with_mock_api PASSED [100%]

============================ 1 passed in ...s ==============================
```

It passes! And it does so almost instantly. The `requests.exceptions.ConnectionError` is gone. We have successfully isolated our test from the network.

The assertions at the end are also very powerful. The `mock_responses` object records all intercepted calls, so we can assert that:
*   Exactly one API call was made (`len(mock_responses.calls) == 1`).
*   The correct URL was called.
*   The response we received was the one we mocked.

### Testing Failure Cases

Mocking is not just for simulating success. It's crucial for testing how your code handles API errors. What if the API returns a 404 Not Found? Our current `GeolocationClient` would raise an exception via `response.raise_for_status()`. Let's test that our main function propagates this error correctly.

```python
# tests/test_user_reporter.py (new test for API failure)
import pytest
import requests

def test_process_user_report_api_error(db_session, mock_responses):
    # Arrange
    db = UserDatabase(db_session)
    mock_responses.get(
        "https://api.geolocation.com/ip/192.168.1.100",
        json={"error": "IP not found"},
        status=404, # Simulate a 404 Not Found error
    )
    api_client = GeolocationClient(base_url="https://api.geolocation.com")

    # Act & Assert
    with pytest.raises(requests.exceptions.HTTPError) as excinfo:
        process_user_report(
            user_id=1,
            db=db,
            api_client=api_client,
            output_dir=DUMMY_OUTPUT_DIR
        )
    
    # Optionally, assert on the error message or status code
    assert "404 Client Error: Not Found" in str(excinfo.value)
```

This test also passes instantly, giving us confidence that our application behaves correctly even when its dependencies fail. This kind of negative testing is extremely difficult and unreliable without mocking.

## Testing File I/O

## Iteration 3: Controlling the Filesystem

We have isolated our database and network interactions. The last remaining dependency is the filesystem. Our function currently writes a report to a directory that we create manually.

```python
# tests/test_user_reporter.py (current state)
DUMMY_OUTPUT_DIR = Path("./dummy_reports")

def test_process_user_report_with_mock_api(...):
    # ...
    DUMMY_OUTPUT_DIR.mkdir(exist_ok=True)
    process_user_report(..., output_dir=DUMMY_OUTPUT_DIR)
    # ...
```

This has several problems:
*   **State Pollution:** The `dummy_reports` directory and its contents are left behind after the test run, cluttering your project.
*   **Test Interference:** If two tests write to the same filename, they can overwrite each other's output, leading to flaky tests.
*   **Assertion Difficulty:** To check the report's contents, we have to manually construct the file path, open it, read it, and then remember to clean it up. This is tedious and error-prone.

### The Problem: Shared Filesystem State

Let's demonstrate the interference problem. Imagine two tests that process reports for the same user ID, but expect different content (perhaps due to different API responses).

```python
# tests/test_file_problem.py
from pathlib import Path
import shutil

# A simplified function that just writes a file
def write_report(user_id: int, content: str, output_dir: Path):
    report_path = output_dir / f"user_report_{user_id}.txt"
    with open(report_path, "w") as f:
        f.write(content)

OUTPUT_DIR = Path("./shared_reports")

def setup_function(function):
    OUTPUT_DIR.mkdir(exist_ok=True)

def teardown_function(function):
    if OUTPUT_DIR.exists():
        shutil.rmtree(OUTPUT_DIR)

def test_report_for_usa():
    user_id = 1
    write_report(user_id, "Country: USA", OUTPUT_DIR)
    
    report_path = OUTPUT_DIR / f"user_report_{user_id}.txt"
    content = report_path.read_text()
    assert "Country: USA" in content

def test_report_for_canada():
    user_id = 1 # Same user ID!
    write_report(user_id, "Country: Canada", OUTPUT_DIR)
    
    report_path = OUTPUT_DIR / f"user_report_{user_id}.txt"
    content = report_path.read_text()
    assert "Country: Canada" in content
```

These tests will pass if run individually. But what if we run them together and a test runner decides to run them in parallel in the future? Or if one test fails after writing the file but before its assertion? The file from one test can be read by another.

Let's simulate a more direct conflict. What if `test_report_for_usa` also checked that the file *doesn't* contain "Canada"?

```python
# tests/test_file_problem_fail.py
# ... (setup is the same as above) ...

def test_report_for_canada_first():
    user_id = 1
    write_report(user_id, "Country: Canada", OUTPUT_DIR)
    # This test passes, but leaves behind a file.

def test_report_for_usa_reads_stale_file():
    user_id = 1
    # Let's assume this test runs second, but the teardown from the first failed.
    # To simulate, we'll just check the state of the file.
    report_path = OUTPUT_DIR / f"user_report_{user_id}.txt"
    content = report_path.read_text()
    assert "Country: USA" not in content # This should pass
    assert "Country: Canada" in content # This will fail!
```

This is a contrived example, but it shows the core issue: when tests share a writable directory, they are no longer independent.

**Root cause identified**: Tests are writing to a hardcoded, shared directory on the filesystem, breaking test isolation.

**Why the current approach can't solve this**: Manual setup and teardown of directories is brittle. If a test fails unexpectedly, the teardown code might not run, leaving artifacts that poison subsequent test runs.

**What we need**: A mechanism that provides each test function with its own unique, empty, temporary directory that is automatically and reliably cleaned up after the test, regardless of whether it passes or fails.

## Working with Temporary Files and Directories

## The Solution: Pytest's `tmp_path` Fixture

Pytest provides a fantastic built-in fixture for this exact problem: `tmp_path`.

The `tmp_path` fixture is a `pathlib.Path` object that points to a unique temporary directory created for each individual test function. Pytest guarantees that this directory will be removed after the test finishes.

### Iteration 3 Solution: Applying `tmp_path`

Using `tmp_path` is incredibly simple. You just add it as an argument to your test function, and pytest provides the path.

Let's write the final, fully isolated version of our test for `process_user_report`. It will use all three of our techniques:
1.  `db_session` for database isolation.
2.  `mock_responses` for network isolation.
3.  `tmp_path` for filesystem isolation.

```python
# tests/test_user_reporter_final.py
from pathlib import Path
import responses
from user_reporter.main import process_user_report
from user_reporter.db import UserDatabase
from user_reporter.api import GeolocationClient

def test_process_user_report_final(db_session, mock_responses, tmp_path):
    # Arrange (Database)
    db = UserDatabase(db_session)

    # Arrange (API Mock)
    mock_responses.get(
        "https://api.geolocation.com/ip/192.168.1.100",
        json={"country": "USA"},
        status=200,
    )
    api_client = GeolocationClient(base_url="https://api.geolocation.com")
    
    # Arrange (Filesystem)
    # `tmp_path` is a Path object to a unique temp directory.
    # We pass it directly to our function.
    output_dir = tmp_path

    # Act
    process_user_report(
        user_id=1,
        db=db,
        api_client=api_client,
        output_dir=output_dir
    )

    # Assert
    # The report file should exist inside the temporary directory.
    expected_report_path = output_dir / "user_report_1.txt"
    assert expected_report_path.exists()

    # We can now safely read the content and assert on it.
    report_content = expected_report_path.read_text()
    assert "User Report for Alice" in report_content
    assert "IP Address: 192.168.1.100" in report_content
    assert "Country: USA" in report_content
```

### Verification

Let's run this final version.

```bash
$ pytest -v tests/test_user_reporter_final.py
=========================== test session starts ============================
...
collected 1 item

tests/test_user_reporter_final.py::test_process_user_report_final PASSED [100%]

============================ 1 passed in ...s ==============================
```

Perfect. The test passes, and we can be confident that it:
*   Ran instantly because it didn't touch the network.
*   Started with a clean, predictable database state.
*   Wrote its output to a safe, temporary location.
*   Left no trace behind after it finished.

This test is fast, reliable, and robust. It is a high-quality unit test that precisely verifies the logic of `process_user_report` without being coupled to the availability or state of its external dependencies.

### `tmp_path` vs `tmpdir`

You may also see an older fixture called `tmpdir`.
*   `tmpdir`: Returns a `py.path.local` object from the `py` library.
*   `tmp_path`: Returns a standard `pathlib.Path` object.

The `pathlib` module is the modern, standard way to handle filesystem paths in Python. **You should always prefer `tmp_path` over `tmpdir` in new code.**

### Synthesis: The Complete Journey

We have taken a complex function with multiple external dependencies and systematically made it testable.

| Iteration | Failure Mode                               | Technique Applied                        | Result                                                              |
| :-------- | :----------------------------------------- | :--------------------------------------- | :------------------------------------------------------------------ |
| 0         | Brittle integration test                   | None                                     | Slow, unreliable, stateful test that pollutes the environment.      |
| 1         | Database state pollution                   | `db_session` fixture with transactions   | Tests are isolated from each other's database changes.              |
| 2         | Network dependency (slowness, flakiness)   | `responses` library and `mock_responses` fixture | Network calls are mocked, making tests fast and deterministic.      |
| 3         | Filesystem pollution & interference        | `tmp_path` built-in fixture              | File I/O is redirected to a clean, temporary, isolated directory.   |

### Final Implementation

Here is the final, testable application code, incorporating the dependency injection patterns we established.

**`user_reporter/db.py`**

```python
import sqlite3
from dataclasses import dataclass

@dataclass
class User:
    id: int
    name: str
    ip_address: str

class UserDatabase:
    def __init__(self, connection: sqlite3.Connection):
        self.conn = connection

    def get_user(self, user_id: int) -> User | None:
        cursor = self.conn.cursor()
        res = cursor.execute(
            "SELECT id, name, ip_address FROM users WHERE id = ?", (user_id,)
        ).fetchone()
        
        if res:
            return User(id=res[0], name=res[1], ip_address=res[2])
        return None
```

**`user_reporter/api.py`**

```python
import requests

class GeolocationClient:
    def __init__(self, base_url="https://api.geolocation.com"):
        self.base_url = base_url

    def get_country_for_ip(self, ip_address: str) -> str:
        response = requests.get(f"{self.base_url}/ip/{ip_address}")
        response.raise_for_status()
        return response.json()["country"]
```

**`user_reporter/main.py`**

```python
from pathlib import Path
from .db import UserDatabase
from .api import GeolocationClient

def process_user_report(
    user_id: int, 
    db: UserDatabase, 
    api_client: GeolocationClient, 
    output_dir: Path
):
    user = db.get_user(user_id)

    if not user:
        raise ValueError(f"User with ID {user_id} not found.")

    country = api_client.get_country_for_ip(user.ip_address)

    report_path = output_dir / f"user_report_{user_id}.txt"
    report_content = f"User Report for {user.name}\n"
    report_content += f"IP Address: {user.ip_address}\n"
    report_content += f"Country: {country}\n"

    with open(report_path, "w") as f:
        f.write(report_content)
```

### Lessons Learned

*   **Isolate Your Tests:** The primary goal when testing code with side effects is to isolate your test from the external dependency. This makes your tests fast, reliable, and deterministic.
*   **Dependency Injection is Key:** The most powerful pattern for enabling testability is Dependency Injection. By passing dependencies (like database connections or API clients) into your functions, you create "seams" where you can substitute real objects with test doubles (fakes, mocks) during testing.
*   **Use the Right Tool for the Job:**
    *   For **databases**, use fixtures to manage connections and transactions for isolation.
    *   For **HTTP requests**, use specialized libraries like `responses` to cleanly mock the network layer.
    *   For the **filesystem**, use pytest's built-in `tmp_path` fixture to get a safe, clean, temporary workspace.
*   **Test the Logic, Not the Framework:** Our final test verifies the business logic of `process_user_report` (that it correctly combines data and formats a string) without actually testing SQLite, the `requests` library, or the operating system's filesystem. We trust that those well-tested components work and focus our tests on our own code.
