# Chapter 12: Testing External Dependencies

## Database Testing Strategies

## Why Testing with Databases is Hard

Your application code rarely lives in isolation. It interacts with the outside world: databases, APIs, file systems, and more. These interactions, or "external dependencies," introduce significant challenges for testing. Unlike pure functions that always produce the same output for the same input, external systems have *state*. A database's content changes, an API might be down, or a file might not exist.

Testing code that interacts with a database is particularly tricky for several reasons:

1.  **Statefulness**: A test that writes data to a database can affect the outcome of the next test that reads from it. This leads to flaky, order-dependent tests, which are a nightmare to maintain.
2.  **Slowness**: Establishing a database connection, creating tables, and running queries is slow. A test suite with hundreds of database interactions can take minutes to run, discouraging developers from running it frequently.
3.  **Complex Setup**: To run tests, you need a database server running, configured with the correct schema, and accessible to the test runner. This complicates both local development and Continuous Integration (CI) environments.

To manage this complexity, developers have adopted several strategies, each with its own set of trade-offs between speed and fidelity.

## Three Core Strategies

There are three primary approaches to testing database-dependent code. The right choice depends on what you are trying to prove with your test.

### Strategy 1: Mocking the Database Layer (Unit Testing)

In this approach, you don't use a database at all. You replace the part of your code that talks to the database (like a repository or a data access object) with a "mock" object.

-   **How it works**: You use a library like `unittest.mock` to create a fake object that mimics the behavior of your database access layer. Your test then configures this mock to return predefined data (e.g., "when `get_user(id=1)` is called, return this fake User object").
-   **Pros**:
    -   **Extremely Fast**: No database connection, no network latency. Tests run in milliseconds.
    -   **Total Isolation**: The test is completely isolated from the database, focusing solely on the business logic of the code under test.
-   **Cons**:
    -   **Low Fidelity**: You are not testing the actual database interaction. Your test will pass even if your SQL query is invalid, if you violate a database constraint, or if your object-relational mapper (ORM) is misconfigured.
    -   **Brittle**: If you change your database schema, you have to remember to update all your mocks, or your tests will become misleading.

This strategy is best for pure **unit tests** where you want to verify business logic without touching the database. We covered mocking in detail in Chapters 8 and 9.

### Strategy 2: Using an In-Memory Database (Integration Testing)

This is a popular middle ground. Instead of connecting to a full-fledged database server like PostgreSQL or MySQL, you use a lightweight, in-memory database like SQLite.

-   **How it works**: For each test run, you create a brand new SQLite database directly in memory. It's incredibly fast to set up and tear down.
-   **Pros**:
    -   **Fast**: Much faster than a real disk-based database. Setup and teardown are nearly instantaneous.
    -   **Good Isolation**: Each test can get its own pristine, empty database, ensuring tests don't interfere with each other.
    -   **High Fidelity for ORMs**: You can test that your ORM (like SQLAlchemy or Django's ORM) correctly generates queries and maps objects.
-   **Cons**:
    -   **Dialect Differences**: In-memory databases like SQLite don't always behave identically to production databases like PostgreSQL. They might have different data types, support different SQL features, or have looser constraint checking. A query that works on SQLite might fail on PostgreSQL.

This strategy is excellent for **integration tests** of your application's data layer, where you want to test your code's interaction with a database-like system without the overhead of a real one.

### Strategy 3: Using a Real Test Database (End-to-End Testing)

This approach offers the highest fidelity. You run your tests against a dedicated instance of the same database software you use in production (e.g., PostgreSQL), but with a separate, temporary database created just for the test suite.

-   **How it works**: Your test setup script (often managed with tools like Docker) spins up a real database instance. Your test suite connects to it, creates the schema, runs the tests, and then tears it all down.
-   **Pros**:
    -   **Maximum Fidelity**: You are testing against the real thing. If it works in the test suite, it will almost certainly work in production. You can test native database features, stored procedures, and complex constraints.
-   **Cons**:
    -   **Slow**: This is the slowest approach by far. Starting a Docker container and setting up a database can take several seconds.
    -   **Complex Setup**: Requires managing a separate database service for testing, which can be complex to configure locally and in CI.

This strategy is best for a smaller set of **end-to-end tests** that verify critical paths of your application against a production-like environment.

In the next section, we'll focus on Strategy 2, using an in-memory SQLite database with pytest fixtures, as it provides a fantastic balance of speed and realism for most development workflows.

## Using Fixtures for Database Setup

## The Problem: Manual Setup and Teardown

Let's imagine we have a simple application that uses SQLAlchemy to manage a `User` model.

First, let's define our application code. We'll need to install SQLAlchemy to run this.

```bash
pip install SQLAlchemy
```

Here is our simple model and a function to create a user.

```python
# src/database.py
from sqlalchemy import (
    create_engine,
    Column,
    Integer,
    String,
)
from sqlalchemy.orm import sessionmaker, declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = "users"
    id = Column(Integer, primary_key=True)
    name = Column(String, nullable=False)
    email = Column(String, unique=True)

    def __repr__(self):
        return f"<User(id={self.id}, name='{self.name}')>"

def create_user(session, name, email):
    """Adds a new user to the database."""
    new_user = User(name=name, email=email)
    session.add(new_user)
    session.commit()
    return new_user
```

Now, how would we test `create_user`? Without fixtures, we might be tempted to write setup and teardown code directly in our test function. This is the "wrong way" that illuminates the need for a better approach.

```python
# tests/test_database_manual.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from src.database import Base, create_user, User

def test_create_user_manual_setup():
    # 1. Setup: Create an in-memory SQLite database
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    Session = sessionmaker(bind=engine)
    session = Session()

    # 2. The actual test
    user = create_user(session, name="Alice", email="alice@example.com")
    assert user.id is not None
    
    retrieved_user = session.query(User).filter_by(name="Alice").first()
    assert retrieved_user.name == "Alice"
    assert retrieved_user.email == "alice@example.com"

    # 3. Teardown: Close the session
    session.close()
    # The in-memory database is automatically discarded
```

This works, but imagine having ten tests like this. You would be copying and pasting the entire setup and teardown block every single time. This violates the DRY (Don't Repeat Yourself) principle and makes the tests harder to read and maintain.

Worse, if we were using a file-based database, a failing test might skip the cleanup step, leaving artifacts that could cause other tests to fail mysteriously. This is where fixtures become essential.

### The Solution: A Fixture for Database Sessions

Let's refactor this using a fixture. A fixture can handle the setup (creating the engine and tables) and the teardown (cleaning up) in one place. We'll place this in `tests/conftest.py` so it's available to all our tests.

```python
# tests/conftest.py
import pytest
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

from src.database import Base

@pytest.fixture(scope="function")
def db_session():
    """
    Fixture to create a new in-memory database session for each test function.
    """
    # Setup: create an in-memory SQLite database
    engine = create_engine("sqlite:///:memory:")
    Base.metadata.create_all(engine)
    Session = sessionmaker(bind=engine)
    session = Session()

    yield session  # This is where the testing happens

    # Teardown: close the session and drop tables
    session.close()
    Base.metadata.drop_all(engine)
```

Let's break down this fixture:

1.  **`@pytest.fixture(scope="function")`**: We define a fixture named `db_session`. The `scope="function"` (the default) is crucial here. It means pytest will run this fixture's setup and teardown code *for each test function that uses it*. This guarantees a clean, empty database for every single test, providing perfect test isolation.
2.  **Setup**: The code before the `yield` is the setup phase. It creates the in-memory engine, creates all tables defined by our `Base` metadata, and creates a session.
3.  **`yield session`**: The `yield` keyword passes control to the test function. The value yielded (`session`) is what gets injected into our test function's argument.
4.  **Teardown**: The code after the `yield` is the teardown phase. It runs after the test function completes, whether it passed, failed, or raised an error. Here, we close the session and drop all tables to be extra clean.

Now, our test becomes beautifully simple and declarative.

```python
# tests/test_database_fixture.py
from src.database import create_user, User

def test_create_user(db_session):
    """
    Given a database session,
    When create_user is called,
    Then a new User should be created in the database.
    """
    # The db_session is provided by our fixture in conftest.py
    user = create_user(db_session, name="Bob", email="bob@example.com")
    
    assert user.id is not None
    
    retrieved_user = db_session.query(User).filter_by(name="Bob").first()
    assert retrieved_user is not None
    assert retrieved_user.name == "Bob"

def test_user_email_uniqueness(db_session):
    """
    Given a user in the database,
    When another user with the same email is created,
    Then an IntegrityError should be raised.
    """
    import sqlalchemy.exc
    
    # Create an initial user
    create_user(db_session, name="Charlie", email="charlie@example.com")
    
    # Try to create another user with the same email
    with pytest.raises(sqlalchemy.exc.IntegrityError):
        create_user(db_session, name="Charles", email="charlie@example.com")
```

Look at the difference!

-   The tests are focused purely on the behavior being tested.
-   There is no repetitive setup/teardown code.
-   We can be certain that `test_create_user` and `test_user_email_uniqueness` run with completely separate, clean databases, thanks to the `function` scope of our fixture.

### Fixture Scopes and Trade-offs

While `scope="function"` provides the best isolation, it can be slow if your schema is large, as it rebuilds the database for every test. You can change the scope to speed things up, but you must manage the trade-offs.

-   **`scope="module"`**: The fixture runs once per test module (file). All tests in that file share the same database connection and data. Faster, but you must manually clean up data created by each test to avoid interference.
-   **`scope="session"`**: The fixture runs once for the entire test session. Extremely fast, but carries a high risk of test interdependency. This is often used for setting up a connection to a real test database (Strategy 3) that persists for the whole run.

For most cases, starting with `scope="function"` is the safest and most reliable choice. Only optimize to a larger scope if database setup becomes a significant bottleneck in your test suite.

## Testing API Calls

## The Challenge of Network-Bound Code

Many applications rely on external APIs. Your code might fetch user data from a third-party service, process a payment through a gateway, or post a message to a chat service. Testing this code presents a major challenge.

Let's consider a simple function that fetches a user's public repositories from the GitHub API using the popular `requests` library.

First, install `requests`:
```bash
pip install requests
```

Now, here's our function:

```python
# src/api_client.py
import requests

class GitHubAPIClient:
    def get_user_repos(self, username):
        """
        Fetches the names of public repositories for a given GitHub user.
        Returns a list of repo names or None if the user is not found.
        """
        if not isinstance(username, str) or not username:
            raise ValueError("Username must be a non-empty string")

        url = f"https://api.github.com/users/{username}/repos"
        response = requests.get(url)

        if response.status_code == 200:
            repos = response.json()
            return [repo["name"] for repo in repos]
        elif response.status_code == 404:
            return None
        else:
            response.raise_for_status() # Raise an exception for other errors
```

How would we test this? We could write a test that calls the real GitHub API.

```python
# tests/test_api_client_wrong.py
from src.api_client import GitHubAPIClient

# DO NOT DO THIS IN A REAL TEST SUITE!
def test_get_user_repos_real_call():
    client = GitHubAPIClient()
    # This test depends on the real state of the 'pytest-dev' user on GitHub
    repos = client.get_user_repos("pytest-dev")
    assert "pytest" in repos
```

This is a terrible idea for an automated test suite. Why?

1.  **Unreliable**: If you have no internet connection, or if GitHub's API is temporarily down, your test will fail. This is a false negative—your code is correct, but the external dependency failed.
2.  **Slow**: The test has to make a real network request across the internet, which can take hundreds of milliseconds or even seconds. A suite of such tests would be painfully slow.
3.  **Brittle**: The test depends on the state of the real world. If the `pytest-dev` organization renames its `pytest` repository, this test will break.
4.  **Rate Limiting**: Many APIs, including GitHub's, have rate limits. Running your test suite frequently could get your IP address temporarily blocked.

The solution is to **mock the HTTP request**. We need to intercept the outgoing call from the `requests` library and feed it a fake, controlled response. This way, our test verifies our code's logic (how it handles a 200 OK, a 404 Not Found, etc.) without ever actually touching the network.

## Mocking HTTP Requests with responses

## A Better Way: The `responses` Library

While you can mock network calls with `unittest.mock.patch`, it can be cumbersome. A far more elegant and powerful tool for this specific job is the `responses` library, which integrates beautifully with pytest via `pytest-responses`.

First, let's install it:
```bash
pip install pytest-responses
```

The `pytest-responses` plugin provides a `responses` fixture that acts as a request-response manager. You tell it which URLs to watch for and what fake responses to return.

### Testing the Success Path

Let's write a proper test for the `get_user_repos` method. We will simulate a successful API call that returns two repositories.

```python
# tests/test_api_client.py
import pytest
from src.api_client import GitHubAPIClient

def test_get_user_repos_success(responses):
    """
    Test fetching user repos successfully.
    """
    # 1. Define the fake response data
    username = "testuser"
    expected_repos = [{"name": "repo1"}, {"name": "repo2"}]
    
    # 2. Register the mock URL with the 'responses' fixture
    responses.add(
        responses.GET,
        f"https://api.github.com/users/{username}/repos",
        json=expected_repos,
        status=200,
    )

    # 3. Call our code
    client = GitHubAPIClient()
    repos = client.get_user_repos(username)

    # 4. Assert the results
    assert repos == ["repo1", "repo2"]
    # Optional: Assert that the mock was called exactly once
    assert len(responses.calls) == 1
    assert responses.calls[0].request.url == f"https://api.github.com/users/{username}/repos"
```

Let's break down this test:

1.  **`test_get_user_repos_success(responses)`**: We request the `responses` fixture, which activates the mocking mechanism for this test.
2.  **`responses.add(...)`**: This is the core of the mock. We are telling the `responses` library: "If you see an HTTP `GET` request to `https://api.github.com/users/testuser/repos`, intercept it. Do not let it go to the internet. Instead, pretend you received a response with a `200` status code and this JSON body."
3.  **`client.get_user_repos(username)`**: When this line executes, `requests.get()` is called internally. The `responses` library intercepts this call because the URL matches our rule. It returns a fake `Response` object with the data we specified.
4.  **Assertions**: We can now assert that our function correctly parsed the fake JSON and returned the list of repository names. We can also inspect `responses.calls` to verify that the expected network call was made.

If our code tried to access any other URL, the `responses` library would raise an error, preventing unexpected network access.

### Testing Failure Paths

Mocking is even more valuable for testing how your code handles errors. It's difficult to reliably trigger a 404 or 500 error from a real API, but with `responses`, it's trivial.

Let's test the "user not found" case.

```python
# tests/test_api_client.py (continued)

def test_get_user_repos_not_found(responses):
    """
    Test handling of a 404 Not Found error.
    """
    username = "nonexistentuser"
    responses.add(
        responses.GET,
        f"https://api.github.com/users/{username}/repos",
        json={"error": "Not Found"},
        status=404,
    )

    client = GitHubAPIClient()
    repos = client.get_user_repos(username)

    assert repos is None
```

Here, we configured the mock to return a `404` status code. Our test verifies that `get_user_repos` correctly handles this by returning `None`, just as we designed it to.

We can also test how our code handles unexpected server errors.

```python
# tests/test_api_client.py (continued)
import requests

def test_get_user_repos_server_error(responses):
    """
    Test handling of a 500 Internal Server Error.
    """
    username = "anyuser"
    responses.add(
        responses.GET,
        f"https://api.github.com/users/{username}/repos",
        status=500,
    )

    client = GitHubAPIClient()
    with pytest.raises(requests.exceptions.HTTPError):
        client.get_user_repos(username)
```

In this test, we simulate a `500` error. Our code is designed to call `response.raise_for_status()` in this case, which should raise an `HTTPError`. Using `pytest.raises`, we can elegantly assert that this expected exception was indeed raised.

By using `responses`, we have created a fast, reliable, and comprehensive test suite for our API client without ever making a single real network request.

## Testing File I/O

## The Perils of a Persistent File System

Just like databases, the file system is a form of external state. Tests that read or write files can interfere with each other and leave a mess on your machine if not handled carefully.

Consider a function that processes a text file, counting the number of lines that contain a specific keyword.

```python
# src/file_processor.py

def analyze_log_file(file_path, keyword):
    """
    Counts the number of lines containing a keyword in a file.
    """
    try:
        with open(file_path, "r") as f:
            lines = f.readlines()
        return sum(1 for line in lines if keyword in line)
    except FileNotFoundError:
        return -1
```

### The Wrong Way: Using Real Files

A naive approach to testing this would be to create a real file on disk.

```python
# tests/test_file_processor_wrong.py
import os

# DO NOT DO THIS!
def test_analyze_log_file_with_real_file():
    file_path = "test_log.txt"
    content = "INFO: Starting process\nWARNING: Low disk space\nINFO: Process finished\n"
    
    # Setup: Create the file
    with open(file_path, "w") as f:
        f.write(content)

    # The actual test
    count = analyze_log_file(file_path, "INFO")
    assert count == 2

    # Teardown: Clean up the file
    os.remove(file_path)
```

This approach is fraught with problems:

1.  **Cleanup is Not Guaranteed**: If the assertion fails, `os.remove(file_path)` is never called. The `test_log.txt` file will be left behind, cluttering your project directory.
2.  **Race Conditions**: If you run tests in parallel (e.g., with `pytest-xdist`), multiple processes might try to write to and delete `test_log.txt` at the same time, causing unpredictable failures.
3.  **Pathing Issues**: Where should `test_log.txt` be created? In the project root? In the `tests/` directory? This can become messy and non-portable.

We need a way to create files and directories in a temporary, isolated location that is automatically and reliably cleaned up after the test finishes.

## Working with Temporary Files and Directories

## Pytest's Built-in `tmp_path` Fixture

Pytest provides a brilliant solution to this problem with its built-in `tmp_path` fixture.

When you add `tmp_path` as an argument to your test function, pytest does the following:

1.  Before the test runs, it creates a unique new temporary directory (e.g., `/tmp/pytest-of-user/pytest-1/test_my_function0/`).
2.  It passes a `pathlib.Path` object pointing to this directory into your test function.
3.  After the test finishes (pass or fail), it recursively removes the entire directory and all its contents.

This gives you a pristine, private sandbox for each test to work with files, guaranteeing isolation and cleanup.

### Refactoring with `tmp_path`

Let's rewrite our test for `analyze_log_file` using `tmp_path`.

```python
# tests/test_file_processor.py
from src.file_processor import analyze_log_file

def test_analyze_log_file_with_tmp_path(tmp_path):
    """
    Given a temporary file created via tmp_path,
    When analyze_log_file is called,
    Then it should return the correct count of lines with the keyword.
    """
    # tmp_path is a pathlib.Path object to a temporary directory
    # 1. Setup: Create a file inside the temporary directory
    log_file = tmp_path / "my_log.txt"
    log_file.write_text("INFO: Starting process\nWARNING: Low disk space\nINFO: Process finished\n")

    # 2. Run the test
    count = analyze_log_file(log_file, "INFO")

    # 3. Assert the result
    assert count == 2

def test_analyze_log_file_not_found(tmp_path):
    """
    Given a path that does not exist,
    When analyze_log_file is called,
    Then it should return -1.
    """
    non_existent_file = tmp_path / "ghost.txt"
    
    count = analyze_log_file(non_existent_file, "ERROR")
    
    assert count == -1
```

This is a massive improvement:

-   **Clean and Declarative**: The test clearly shows its intent. It creates a file, calls the function, and checks the result.
-   **No Manual Cleanup**: We don't need any `try...finally` blocks or `os.remove()` calls. Pytest handles it all.
-   **`pathlib` Power**: `tmp_path` is a `pathlib.Path` object, which provides a modern, object-oriented API for file system operations (`/` for joining paths, `.write_text()`, `.read_text()`, etc.).
-   **Complete Isolation**: Each test function gets its own unique `tmp_path`, so there is zero chance of interference.

### Testing Functions That Write Files

The `tmp_path` fixture is also perfect for testing functions that *create* files. Let's imagine a function that generates a report.

```python
# src/file_processor.py (continued)

def generate_report(output_path, data):
    """
    Writes a simple report to the given output path.
    """
    with open(output_path, "w") as f:
        f.write("--- REPORT ---\n")
        for key, value in data.items():
            f.write(f"{key}: {value}\n")
        f.write("--- END ---\n")
```

Our test can use `tmp_path` to provide a safe output location and then verify the contents of the created file.

```python
# tests/test_file_processor.py (continued)
from src.file_processor import generate_report

def test_generate_report(tmp_path):
    """
    Given a temporary output path,
    When generate_report is called,
    Then it should create a file with the correct content.
    """
    report_file = tmp_path / "report.txt"
    data = {"user_count": 150, "status": "OK"}

    generate_report(report_file, data)

    # Assert that the file was created and has the correct content
    assert report_file.exists()
    
    content = report_file.read_text()
    assert "--- REPORT ---" in content
    assert "user_count: 150" in content
    assert "status: OK" in content
    assert "--- END ---" in content
```

By using fixtures like `db_session` for databases, `responses` for APIs, and `tmp_path` for files, you can tame the complexity of external dependencies. These tools allow you to write tests that are fast, reliable, and isolated, forming the bedrock of a robust and maintainable test suite.
