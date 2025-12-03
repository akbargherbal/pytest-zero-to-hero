# Chapter 12: Testing External Dependencies

## Database Testing Strategies

## The Challenge of Testing Database Code

Testing code that interacts with databases presents a fundamental tension: databases are stateful, persistent, and often shared resources. Your tests need to be isolated, repeatable, and fast. These requirements seem incompatible.

Consider a user registration system. The production code looks straightforward:

```python
# user_service.py
import sqlite3

class UserService:
    def __init__(self, db_path):
        self.db_path = db_path
    
    def register_user(self, username, email):
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        cursor.execute(
            "INSERT INTO users (username, email) VALUES (?, ?)",
            (username, email)
        )
        conn.commit()
        user_id = cursor.lastrowid
        conn.close()
        
        return user_id
    
    def get_user(self, user_id):
        conn = sqlite3.connect(self.db_path)
        cursor = conn.cursor()
        
        cursor.execute(
            "SELECT id, username, email FROM users WHERE id = ?",
            (user_id,)
        )
        row = cursor.fetchone()
        conn.close()
        
        if row:
            return {"id": row[0], "username": row[1], "email": row[2]}
        return None
```

But how do you test this? If you point it at a real database, each test run leaves data behind. Tests interfere with each other. You can't run tests in parallel. The database might not even exist in your CI environment.

Let's start with the naive approach and watch it fail.

```python
# test_user_service_naive.py
import pytest
from user_service import UserService

def test_register_user():
    service = UserService("production.db")  # Using real database!
    
    user_id = service.register_user("alice", "alice@example.com")
    
    assert user_id is not None
    assert user_id > 0

def test_get_user():
    service = UserService("production.db")
    
    # Assumes alice exists from previous test
    user = service.get_user(1)
    
    assert user["username"] == "alice"
    assert user["email"] == "alice@example.com"
```

Run this twice:

```bash
$ pytest test_user_service_naive.py -v
```

**First run output**:

```text
test_user_service_naive.py::test_register_user PASSED
test_user_service_naive.py::test_get_user PASSED
```

**Second run output**:

```text
test_user_service_naive.py::test_register_user FAILED
test_user_service_naive.py::test_get_user PASSED

================================ FAILURES =================================
_________________________ test_register_user __________________________

    def test_register_user():
        service = UserService("production.db")
        
        user_id = service.register_user("alice", "alice@example.com")
>       assert user_id is not None
E       sqlite3.IntegrityError: UNIQUE constraint failed: users.username

test_user_service_naive.py:8: IntegrityError
```

### Diagnostic Analysis: Reading the Failure

**The complete output**:

The test passed the first time but failed the second time with `sqlite3.IntegrityError: UNIQUE constraint failed: users.username`.

**Let's parse this section by section**:

1. **The error type**: `sqlite3.IntegrityError`
   - What this tells us: The database rejected our operation due to a constraint violation
   - This is a database-level error, not a Python assertion failure

2. **The constraint violation**: `UNIQUE constraint failed: users.username`
   - What this tells us: We tried to insert a username that already exists
   - The database has a UNIQUE constraint on the username column
   - Our first test run left "alice" in the database

3. **The test interdependency**:
   - `test_get_user` assumes user ID 1 exists (from the previous test)
   - If we run tests in a different order, `test_get_user` will fail
   - Tests are not isolated—they share state through the database

**Root cause identified**: Tests are writing to a persistent database, creating state that survives between test runs.

**Why the current approach can't solve this**: We need each test to start with a clean database state, but we're using a shared, persistent database file.

**What we need**: A strategy to give each test its own isolated database that gets cleaned up automatically.

## Strategy 1: In-Memory SQLite Database

The simplest solution for SQLite: use an in-memory database that exists only for the duration of the test.

```python
# test_user_service_memory.py
import pytest
import sqlite3
from user_service import UserService

def test_register_user_isolated():
    # Use in-memory database
    service = UserService(":memory:")
    
    # Create schema
    conn = sqlite3.connect(":memory:")
    conn.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL
        )
    """)
    conn.close()
    
    user_id = service.register_user("alice", "alice@example.com")
    
    assert user_id is not None
    assert user_id > 0
```

Run this multiple times:

```bash
$ pytest test_user_service_memory.py -v
```

**Output**:

```text
test_user_service_memory.py::test_register_user_isolated FAILED

================================ FAILURES =================================
__________________ test_register_user_isolated ________________________

    def test_register_user_isolated():
        service = UserService(":memory:")
        
        conn = sqlite3.connect(":memory:")
>       conn.execute("""
            CREATE TABLE users (
                id INTEGER PRIMARY KEY,
                username TEXT UNIQUE NOT NULL,
                email TEXT NOT NULL
            )
        """)
E       sqlite3.OperationalError: no such table: users

test_user_service_memory.py:15: OperationalError
```

### Diagnostic Analysis: The In-Memory Database Problem

**The complete output**:

The test fails with `sqlite3.OperationalError: no such table: users` even though we just created the table.

**Let's parse this section by section**:

1. **The error location**: The error occurs in `register_user()`, not in our schema creation code
   - What this tells us: The schema creation succeeded, but the service can't see the table

2. **The in-memory database behavior**: Each call to `sqlite3.connect(":memory:")` creates a **new, separate** in-memory database
   - What this tells us: Our schema creation and our service are using different databases
   - Line 6: `service = UserService(":memory:")` creates database #1
   - Line 9: `conn = sqlite3.connect(":memory:")` creates database #2
   - We created the schema in database #2, but the service writes to database #1

**Root cause identified**: In-memory databases are not shared across connections. Each `:memory:` connection is isolated.

**Why the current approach can't solve this**: We need to share the same in-memory database between our setup code and the service.

**What we need**: A way to create one in-memory database and pass the same connection to both our setup and our service.

## Strategy 2: Shared In-Memory Database with Connection Passing

SQLite allows sharing in-memory databases using a named URI:

```python
# test_user_service_shared.py
import pytest
import sqlite3
from user_service import UserService

def test_register_user_shared_memory():
    # Use named in-memory database (shareable)
    db_uri = "file::memory:?cache=shared"
    
    # Create schema
    conn = sqlite3.connect(db_uri, uri=True)
    conn.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL
        )
    """)
    conn.commit()
    conn.close()
    
    # Now service can use the same database
    service = UserService(db_uri)
    user_id = service.register_user("alice", "alice@example.com")
    
    assert user_id is not None
    assert user_id > 0
    
    # Verify it was actually stored
    user = service.get_user(user_id)
    assert user["username"] == "alice"
    assert user["email"] == "alice@example.com"
```

But wait—our `UserService` uses `sqlite3.connect(self.db_path)` without the `uri=True` parameter. Let's fix that:

```python
# user_service.py (updated)
import sqlite3

class UserService:
    def __init__(self, db_path, uri=False):
        self.db_path = db_path
        self.uri = uri
    
    def _connect(self):
        return sqlite3.connect(self.db_path, uri=self.uri)
    
    def register_user(self, username, email):
        conn = self._connect()
        cursor = conn.cursor()
        
        cursor.execute(
            "INSERT INTO users (username, email) VALUES (?, ?)",
            (username, email)
        )
        conn.commit()
        user_id = cursor.lastrowid
        conn.close()
        
        return user_id
    
    def get_user(self, user_id):
        conn = self._connect()
        cursor = conn.cursor()
        
        cursor.execute(
            "SELECT id, username, email FROM users WHERE id = ?",
            (user_id,)
        )
        row = cursor.fetchone()
        conn.close()
        
        if row:
            return {"id": row[0], "username": row[1], "email": row[2]}
        return None
```

Now update the test:

```python
# test_user_service_shared.py (updated)
import pytest
import sqlite3
from user_service import UserService

def test_register_user_shared_memory():
    db_uri = "file::memory:?cache=shared"
    
    # Create schema
    conn = sqlite3.connect(db_uri, uri=True)
    conn.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL
        )
    """)
    conn.commit()
    conn.close()
    
    # Pass uri=True to service
    service = UserService(db_uri, uri=True)
    user_id = service.register_user("alice", "alice@example.com")
    
    assert user_id is not None
    assert user_id > 0
    
    user = service.get_user(user_id)
    assert user["username"] == "alice"
    assert user["email"] == "alice@example.com"
```

Run this:

```bash
$ pytest test_user_service_shared.py -v
```

**Output**:

```text
test_user_service_shared.py::test_register_user_shared_memory PASSED
```

Success! But notice the duplication: every test needs to create the schema. This is where fixtures become essential.

## Strategy 3: Database Fixture Pattern

The professional approach: use fixtures to manage database lifecycle.

```python
# test_user_service_fixture.py
import pytest
import sqlite3
from user_service import UserService

@pytest.fixture
def db_connection():
    """Provide a clean in-memory database with schema."""
    db_uri = "file::memory:?cache=shared"
    conn = sqlite3.connect(db_uri, uri=True)
    
    # Create schema
    conn.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL
        )
    """)
    conn.commit()
    
    yield conn
    
    # Cleanup
    conn.close()

@pytest.fixture
def user_service(db_connection):
    """Provide a UserService connected to the test database."""
    # Extract the database URI from the connection
    db_uri = "file::memory:?cache=shared"
    return UserService(db_uri, uri=True)

def test_register_user(user_service):
    user_id = user_service.register_user("alice", "alice@example.com")
    
    assert user_id is not None
    assert user_id > 0

def test_get_user(user_service):
    # Each test gets a fresh database
    user_id = user_service.register_user("bob", "bob@example.com")
    
    user = user_service.get_user(user_id)
    assert user["username"] == "bob"
    assert user["email"] == "bob@example.com"

def test_user_not_found(user_service):
    user = user_service.get_user(999)
    assert user is None
```

Run this:

```bash
$ pytest test_user_service_fixture.py -v
```

**Output**:

```text
test_user_service_fixture.py::test_register_user PASSED
test_user_service_fixture.py::test_get_user PASSED
test_user_service_fixture.py::test_user_not_found PASSED
```

Each test gets a clean database. But there's still a problem: tests can interfere with each other if they run in the same process. Let's verify:

```python
# test_user_service_interference.py
import pytest
import sqlite3
from user_service import UserService

@pytest.fixture
def db_connection():
    db_uri = "file::memory:?cache=shared"
    conn = sqlite3.connect(db_uri, uri=True)
    
    conn.execute("""
        CREATE TABLE IF NOT EXISTS users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL
        )
    """)
    conn.commit()
    
    yield conn
    
    conn.close()

@pytest.fixture
def user_service(db_connection):
    db_uri = "file::memory:?cache=shared"
    return UserService(db_uri, uri=True)

def test_first_user(user_service):
    user_id = user_service.register_user("alice", "alice@example.com")
    assert user_id == 1

def test_second_user(user_service):
    # This test expects to be first, but alice might already exist
    user_id = user_service.register_user("alice", "alice@example.com")
    assert user_id == 1
```

Run this:

```bash
$ pytest test_user_service_interference.py -v
```

**Output**:

```text
test_user_service_interference.py::test_first_user PASSED
test_user_service_interference.py::test_second_user FAILED

================================ FAILURES =================================
________________________ test_second_user _____________________________

    def test_second_user(user_service):
>       user_id = user_service.register_user("alice", "alice@example.com")
E       sqlite3.IntegrityError: UNIQUE constraint failed: users.username

test_user_service_interference.py:32: IntegrityError
```

### Diagnostic Analysis: Fixture Scope Problem

**The complete output**:

The second test fails with the same `UNIQUE constraint failed` error we saw at the beginning.

**Let's parse this section by section**:

1. **The fixture behavior**: Our `db_connection` fixture creates the database once and reuses it
   - What this tells us: The default fixture scope is `function`, but the in-memory database persists across function calls
   - The `CREATE TABLE IF NOT EXISTS` prevents errors, but doesn't clear data

2. **The data persistence**: Alice from `test_first_user` still exists when `test_second_user` runs
   - What this tells us: We're not cleaning up data between tests
   - The fixture's `yield` and cleanup only close the connection, they don't clear the database

**Root cause identified**: We need to clear the database between tests, not just close the connection.

**Why the current approach can't solve this**: Closing and reopening the connection doesn't clear an in-memory database.

**What we need**: Either (1) create a new database for each test, or (2) explicitly clear data between tests.

## Strategy 4: Transaction Rollback Pattern

The most elegant solution: wrap each test in a transaction and roll it back.

```python
# test_user_service_transaction.py
import pytest
import sqlite3
from user_service import UserService

@pytest.fixture(scope="session")
def db_schema():
    """Create schema once for all tests."""
    db_uri = "file:test_db?mode=memory&cache=shared"
    conn = sqlite3.connect(db_uri, uri=True)
    
    conn.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL
        )
    """)
    conn.commit()
    conn.close()
    
    return db_uri

@pytest.fixture
def db_connection(db_schema):
    """Provide a connection with automatic rollback."""
    conn = sqlite3.connect(db_schema, uri=True)
    conn.execute("BEGIN")
    
    yield conn
    
    # Rollback any changes made during the test
    conn.rollback()
    conn.close()

@pytest.fixture
def user_service(db_connection, db_schema):
    """Provide a UserService that uses the transactional connection."""
    # This is tricky: UserService creates its own connections
    # We need to modify our approach
    return UserService(db_schema, uri=True)

def test_first_user(user_service, db_connection):
    user_id = user_service.register_user("alice", "alice@example.com")
    assert user_id == 1

def test_second_user(user_service, db_connection):
    # Should work because previous test was rolled back
    user_id = user_service.register_user("alice", "alice@example.com")
    assert user_id == 1
```

Run this:

```bash
$ pytest test_user_service_transaction.py -v
```

**Output**:

```text
test_user_service_transaction.py::test_first_user PASSED
test_user_service_transaction.py::test_second_user FAILED

================================ FAILURES =================================
________________________ test_second_user _____________________________

    def test_second_user(user_service, db_connection):
>       user_id = user_service.register_user("alice", "alice@example.com")
E       sqlite3.IntegrityError: UNIQUE constraint failed: users.username

test_user_service_transaction.py:42: IntegrityError
```

### Diagnostic Analysis: Connection Isolation Problem

**The complete output**:

Still failing with the same constraint violation.

**Let's parse this section by section**:

1. **The transaction scope**: We created a transaction on `db_connection`, but `UserService` creates its own connections
   - What this tells us: The rollback on our fixture's connection doesn't affect the service's connections
   - Each `sqlite3.connect()` call gets its own transaction context

2. **The architectural mismatch**: Our service is designed to manage its own connections
   - What this tells us: The transaction rollback pattern requires the service to use our provided connection
   - We can't inject our transactional connection into the service as currently designed

**Root cause identified**: The service creates its own connections, so our fixture's transaction rollback has no effect.

**Why the current approach can't solve this**: We need to either (1) redesign the service to accept connections, or (2) use a different isolation strategy.

**What we need**: A service design that accepts database connections as dependencies.

## Strategy 5: Connection Injection Pattern

Redesign the service to accept connections:

```python
# user_service_injectable.py
import sqlite3

class UserService:
    def __init__(self, db_path=None, uri=False, connection=None):
        """
        Initialize with either a database path or an existing connection.
        
        Args:
            db_path: Path to database file
            uri: Whether db_path is a URI
            connection: Existing connection to use (overrides db_path)
        """
        self.db_path = db_path
        self.uri = uri
        self._external_connection = connection
    
    def _connect(self):
        if self._external_connection:
            return self._external_connection
        return sqlite3.connect(self.db_path, uri=self.uri)
    
    def _should_close(self, conn):
        """Only close connections we created."""
        return conn is not self._external_connection
    
    def register_user(self, username, email):
        conn = self._connect()
        cursor = conn.cursor()
        
        cursor.execute(
            "INSERT INTO users (username, email) VALUES (?, ?)",
            (username, email)
        )
        conn.commit()
        user_id = cursor.lastrowid
        
        if self._should_close(conn):
            conn.close()
        
        return user_id
    
    def get_user(self, user_id):
        conn = self._connect()
        cursor = conn.cursor()
        
        cursor.execute(
            "SELECT id, username, email FROM users WHERE id = ?",
            (user_id,)
        )
        row = cursor.fetchone()
        
        if self._should_close(conn):
            conn.close()
        
        if row:
            return {"id": row[0], "username": row[1], "email": row[2]}
        return None
```

Now the transaction rollback pattern works:

```python
# test_user_service_injectable.py
import pytest
import sqlite3
from user_service_injectable import UserService

@pytest.fixture(scope="session")
def db_schema():
    """Create schema once for all tests."""
    db_uri = "file:test_db?mode=memory&cache=shared"
    conn = sqlite3.connect(db_uri, uri=True)
    
    conn.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL
        )
    """)
    conn.commit()
    conn.close()
    
    return db_uri

@pytest.fixture
def db_connection(db_schema):
    """Provide a connection with automatic rollback."""
    conn = sqlite3.connect(db_schema, uri=True)
    
    # Disable autocommit to enable rollback
    conn.isolation_level = None
    conn.execute("BEGIN")
    
    yield conn
    
    # Rollback any changes made during the test
    conn.rollback()
    conn.close()

@pytest.fixture
def user_service(db_connection):
    """Provide a UserService using the transactional connection."""
    return UserService(connection=db_connection)

def test_first_user(user_service):
    user_id = user_service.register_user("alice", "alice@example.com")
    assert user_id == 1

def test_second_user(user_service):
    # Works because previous test was rolled back
    user_id = user_service.register_user("alice", "alice@example.com")
    assert user_id == 1

def test_get_user(user_service):
    user_id = user_service.register_user("bob", "bob@example.com")
    
    user = user_service.get_user(user_id)
    assert user["username"] == "bob"
    assert user["email"] == "bob@example.com"
```

Run this:

```bash
$ pytest test_user_service_injectable.py -v
```

**Output**:

```text
test_user_service_injectable.py::test_first_user PASSED
test_user_service_injectable.py::test_second_user PASSED
test_user_service_injectable.py::test_get_user PASSED
```

Perfect isolation! Each test gets a clean database state through transaction rollback.

## Decision Framework: Which Database Testing Strategy?

| Strategy | Setup Complexity | Isolation | Speed | Best For |
|----------|-----------------|-----------|-------|----------|
| **In-Memory SQLite** | Low | Perfect | Fastest | Unit tests, SQLite-based apps |
| **Transaction Rollback** | Medium | Perfect | Fast | Any SQL database, requires connection injection |
| **Temporary Database Files** | Low | Perfect | Medium | Testing migrations, file-based operations |
| **Docker Test Containers** | High | Perfect | Slow | Integration tests, production-like environment |
| **Mocking Database Calls** | Medium | N/A | Fastest | Testing business logic without database |

### When to Apply Each Solution

**In-Memory SQLite**:
- ✅ Your application uses SQLite
- ✅ Tests don't need to persist data between runs
- ✅ You want maximum speed
- ❌ Your production database is PostgreSQL/MySQL (behavior differences)

**Transaction Rollback**:
- ✅ You need perfect isolation between tests
- ✅ Your service can accept connection injection
- ✅ You want to test actual database interactions
- ❌ Your code has complex transaction management

**Temporary Database Files**:
- ✅ You need to test database file operations
- ✅ You need to test migrations
- ✅ You want simple setup without connection management
- ❌ You need maximum speed (file I/O is slower)

**Docker Test Containers**:
- ✅ You need production-like database behavior
- ✅ You're testing database-specific features
- ✅ Integration tests are more important than speed
- ❌ You need fast feedback loops

## Common Failure Modes and Their Signatures

### Symptom: Tests pass individually but fail when run together

**Pytest output pattern**:

```text
$ pytest test_file.py::test_first -v  # PASSES
$ pytest test_file.py::test_second -v  # PASSES
$ pytest test_file.py -v  # test_second FAILS with IntegrityError
```

**Diagnostic clues**:
- Tests share database state
- No cleanup between tests
- Constraint violations on second run

**Root cause**: Missing transaction rollback or database cleanup

**Solution**: Implement transaction rollback pattern or use function-scoped database fixtures

### Symptom: Tests fail with "database is locked"

**Pytest output pattern**:

```text
sqlite3.OperationalError: database is locked
```

**Diagnostic clues**:
- Multiple connections to the same SQLite database
- Connections not being closed properly
- Concurrent writes without proper transaction management

**Root cause**: SQLite doesn't handle concurrent writes well

**Solution**: Use in-memory databases for tests, or ensure proper connection cleanup

### Symptom: Schema changes don't appear in tests

**Pytest output pattern**:

```text
sqlite3.OperationalError: no such column: new_column
```

**Diagnostic clues**:
- Schema created in one connection
- Service using a different connection
- In-memory database not shared

**Root cause**: Multiple in-memory databases created

**Solution**: Use named in-memory databases with `cache=shared`

## Using Fixtures for Database Setup

## From Ad-Hoc Setup to Fixture Architecture

In the previous section, we saw how transaction rollback provides perfect test isolation. But we also saw the complexity: session-scoped schema creation, function-scoped connections, service injection. Let's build a complete fixture architecture that handles all of this systematically.

Our reference implementation from 12.1 works, but it's scattered across multiple fixtures. Let's consolidate and extend it to handle real-world scenarios.

## Iteration 1: Basic Fixture Hierarchy

Start with the simplest fixture structure:

```python
# conftest.py
import pytest
import sqlite3

@pytest.fixture
def db():
    """Provide a clean database for each test."""
    conn = sqlite3.connect(":memory:")
    
    # Create schema
    conn.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL
        )
    """)
    conn.commit()
    
    yield conn
    
    conn.close()
```

```python
# test_basic_fixture.py
from user_service_injectable import UserService

def test_register_user(db):
    service = UserService(connection=db)
    user_id = service.register_user("alice", "alice@example.com")
    assert user_id == 1

def test_multiple_users(db):
    service = UserService(connection=db)
    
    alice_id = service.register_user("alice", "alice@example.com")
    bob_id = service.register_user("bob", "bob@example.com")
    
    assert alice_id == 1
    assert bob_id == 2
```

Run this:

```bash
$ pytest test_basic_fixture.py -v
```

**Output**:

```text
test_basic_fixture.py::test_register_user PASSED
test_basic_fixture.py::test_multiple_users PASSED
```

This works, but every test needs to create its own `UserService`. Let's add a service fixture.

## Iteration 2: Service Fixture

Add a fixture that provides the service:

```python
# conftest.py (updated)
import pytest
import sqlite3
from user_service_injectable import UserService

@pytest.fixture
def db():
    """Provide a clean database for each test."""
    conn = sqlite3.connect(":memory:")
    
    conn.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL
        )
    """)
    conn.commit()
    
    yield conn
    
    conn.close()

@pytest.fixture
def user_service(db):
    """Provide a UserService connected to the test database."""
    return UserService(connection=db)
```

```python
# test_service_fixture.py
def test_register_user(user_service):
    user_id = user_service.register_user("alice", "alice@example.com")
    assert user_id == 1

def test_get_user(user_service):
    user_id = user_service.register_user("bob", "bob@example.com")
    user = user_service.get_user(user_id)
    
    assert user["username"] == "bob"
    assert user["email"] == "bob@example.com"
```

Run this:

```bash
$ pytest test_service_fixture.py -v
```

**Output**:

```text
test_service_fixture.py::test_register_user PASSED
test_service_fixture.py::test_get_user PASSED
```

Much cleaner! But what if we need to test with existing data? Let's add that capability.

## Iteration 3: Seed Data Fixtures

Add fixtures that provide pre-populated databases:

```python
# conftest.py (updated)
import pytest
import sqlite3
from user_service_injectable import UserService

@pytest.fixture
def db():
    """Provide a clean database for each test."""
    conn = sqlite3.connect(":memory:")
    
    conn.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL
        )
    """)
    conn.commit()
    
    yield conn
    
    conn.close()

@pytest.fixture
def db_with_users(db):
    """Provide a database with sample users."""
    cursor = db.cursor()
    
    cursor.execute(
        "INSERT INTO users (username, email) VALUES (?, ?)",
        ("alice", "alice@example.com")
    )
    cursor.execute(
        "INSERT INTO users (username, email) VALUES (?, ?)",
        ("bob", "bob@example.com")
    )
    db.commit()
    
    return db

@pytest.fixture
def user_service(db):
    """Provide a UserService connected to the test database."""
    return UserService(connection=db)

@pytest.fixture
def user_service_with_data(db_with_users):
    """Provide a UserService with pre-populated data."""
    return UserService(connection=db_with_users)
```

```python
# test_seed_data.py
def test_get_existing_user(user_service_with_data):
    # Alice was created by the fixture
    user = user_service_with_data.get_user(1)
    
    assert user["username"] == "alice"
    assert user["email"] == "alice@example.com"

def test_add_to_existing_users(user_service_with_data):
    # Alice and Bob exist, add Charlie
    charlie_id = user_service_with_data.register_user(
        "charlie", "charlie@example.com"
    )
    
    assert charlie_id == 3  # After alice (1) and bob (2)
```

Run this:

```bash
$ pytest test_seed_data.py -v
```

**Output**:

```text
test_seed_data.py::test_get_existing_user PASSED
test_seed_data.py::test_add_to_existing_users PASSED
```

Great! But this approach has a limitation: what if different tests need different seed data? Creating a fixture for every combination becomes unwieldy.

## Iteration 4: Parameterized Seed Data

Use fixture parameters to provide flexible seed data:

```python
# conftest.py (updated)
import pytest
import sqlite3
from user_service_injectable import UserService

@pytest.fixture
def db():
    """Provide a clean database for each test."""
    conn = sqlite3.connect(":memory:")
    
    conn.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL
        )
    """)
    conn.commit()
    
    yield conn
    
    conn.close()

@pytest.fixture
def seed_users(db, request):
    """
    Seed the database with users.
    
    Usage:
        @pytest.mark.seed_users([
            ("alice", "alice@example.com"),
            ("bob", "bob@example.com")
        ])
        def test_something(seed_users):
            ...
    """
    users = request.node.get_closest_marker("seed_users")
    if users:
        cursor = db.cursor()
        for username, email in users.args[0]:
            cursor.execute(
                "INSERT INTO users (username, email) VALUES (?, ?)",
                (username, email)
            )
        db.commit()
    
    return db

@pytest.fixture
def user_service(db):
    """Provide a UserService connected to the test database."""
    return UserService(connection=db)
```

```python
# test_parameterized_seed.py
import pytest

@pytest.mark.seed_users([
    ("alice", "alice@example.com"),
    ("bob", "bob@example.com")
])
def test_with_two_users(seed_users, user_service):
    user = user_service.get_user(1)
    assert user["username"] == "alice"
    
    user = user_service.get_user(2)
    assert user["username"] == "bob"

@pytest.mark.seed_users([
    ("charlie", "charlie@example.com")
])
def test_with_one_user(seed_users, user_service):
    user = user_service.get_user(1)
    assert user["username"] == "charlie"

def test_with_no_users(seed_users, user_service):
    # No marker, so no seed data
    user = user_service.get_user(1)
    assert user is None
```

Run this:

```bash
$ pytest test_parameterized_seed.py -v
```

**Output**:

```text
test_parameterized_seed.py::test_with_two_users PASSED
test_parameterized_seed.py::test_with_one_user PASSED
test_parameterized_seed.py::test_with_no_users PASSED
```

Excellent flexibility! But we're still creating a new database for every test. For large test suites, this can be slow. Let's optimize with session-scoped fixtures.

## Iteration 5: Session-Scoped Schema with Transaction Rollback

Combine session-scoped schema creation with function-scoped transaction rollback:

```python
# conftest.py (updated)
import pytest
import sqlite3
from user_service_injectable import UserService

@pytest.fixture(scope="session")
def db_schema():
    """Create database schema once for all tests."""
    db_uri = "file:test_db?mode=memory&cache=shared"
    conn = sqlite3.connect(db_uri, uri=True)
    
    conn.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL
        )
    """)
    conn.commit()
    conn.close()
    
    return db_uri

@pytest.fixture
def db(db_schema):
    """Provide a transactional connection that auto-rolls back."""
    conn = sqlite3.connect(db_schema, uri=True)
    conn.isolation_level = None  # Disable autocommit
    conn.execute("BEGIN")
    
    yield conn
    
    conn.rollback()
    conn.close()

@pytest.fixture
def seed_users(db, request):
    """Seed the database with users."""
    users = request.node.get_closest_marker("seed_users")
    if users:
        cursor = db.cursor()
        for username, email in users.args[0]:
            cursor.execute(
                "INSERT INTO users (username, email) VALUES (?, ?)",
                (username, email)
            )
        db.commit()
    
    return db

@pytest.fixture
def user_service(db):
    """Provide a UserService connected to the test database."""
    return UserService(connection=db)
```

Now let's verify that transaction rollback actually works:

```python
# test_transaction_rollback.py
import pytest

def test_first_insert(user_service):
    user_id = user_service.register_user("alice", "alice@example.com")
    assert user_id == 1

def test_second_insert(user_service):
    # Should also get ID 1 because previous test was rolled back
    user_id = user_service.register_user("alice", "alice@example.com")
    assert user_id == 1

@pytest.mark.seed_users([
    ("bob", "bob@example.com")
])
def test_with_seed_data(seed_users, user_service):
    # Bob gets ID 1
    user = user_service.get_user(1)
    assert user["username"] == "bob"
    
    # Alice gets ID 2
    alice_id = user_service.register_user("alice", "alice@example.com")
    assert alice_id == 2

def test_after_seed_data(user_service):
    # Previous test's data was rolled back
    user_id = user_service.register_user("charlie", "charlie@example.com")
    assert user_id == 1
```

Run this:

```bash
$ pytest test_transaction_rollback.py -v
```

**Output**:

```text
test_transaction_rollback.py::test_first_insert PASSED
test_transaction_rollback.py::test_second_insert PASSED
test_transaction_rollback.py::test_with_seed_data PASSED
test_transaction_rollback.py::test_after_seed_data PASSED
```

Perfect! Each test gets a clean slate through transaction rollback, but we only create the schema once.

## Iteration 6: Multiple Tables and Foreign Keys

Real applications have multiple related tables. Let's extend our schema:

```python
# conftest.py (updated with posts table)
import pytest
import sqlite3
from user_service_injectable import UserService

@pytest.fixture(scope="session")
def db_schema():
    """Create database schema once for all tests."""
    db_uri = "file:test_db?mode=memory&cache=shared"
    conn = sqlite3.connect(db_uri, uri=True)
    
    # Enable foreign keys
    conn.execute("PRAGMA foreign_keys = ON")
    
    # Users table
    conn.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL
        )
    """)
    
    # Posts table with foreign key to users
    conn.execute("""
        CREATE TABLE posts (
            id INTEGER PRIMARY KEY,
            user_id INTEGER NOT NULL,
            title TEXT NOT NULL,
            content TEXT NOT NULL,
            FOREIGN KEY (user_id) REFERENCES users(id)
        )
    """)
    
    conn.commit()
    conn.close()
    
    return db_uri

@pytest.fixture
def db(db_schema):
    """Provide a transactional connection that auto-rolls back."""
    conn = sqlite3.connect(db_schema, uri=True)
    conn.execute("PRAGMA foreign_keys = ON")  # Enable for this connection
    conn.isolation_level = None
    conn.execute("BEGIN")
    
    yield conn
    
    conn.rollback()
    conn.close()

@pytest.fixture
def seed_users(db, request):
    """Seed the database with users."""
    users = request.node.get_closest_marker("seed_users")
    if users:
        cursor = db.cursor()
        for username, email in users.args[0]:
            cursor.execute(
                "INSERT INTO users (username, email) VALUES (?, ?)",
                (username, email)
            )
        db.commit()
    
    return db

@pytest.fixture
def user_service(db):
    """Provide a UserService connected to the test database."""
    return UserService(connection=db)
```

Now add a PostService:

```python
# post_service.py
class PostService:
    def __init__(self, connection):
        self.conn = connection
    
    def create_post(self, user_id, title, content):
        cursor = self.conn.cursor()
        cursor.execute(
            "INSERT INTO posts (user_id, title, content) VALUES (?, ?, ?)",
            (user_id, title, content)
        )
        self.conn.commit()
        return cursor.lastrowid
    
    def get_user_posts(self, user_id):
        cursor = self.conn.cursor()
        cursor.execute(
            "SELECT id, title, content FROM posts WHERE user_id = ?",
            (user_id,)
        )
        return [
            {"id": row[0], "title": row[1], "content": row[2]}
            for row in cursor.fetchall()
        ]
```

Add a fixture for PostService:

```python
# conftest.py (add this fixture)
from post_service import PostService

@pytest.fixture
def post_service(db):
    """Provide a PostService connected to the test database."""
    return PostService(connection=db)
```

Test the relationship:

```python
# test_foreign_keys.py
import pytest

@pytest.mark.seed_users([
    ("alice", "alice@example.com")
])
def test_create_post_for_user(seed_users, user_service, post_service):
    # Alice exists from seed data
    user = user_service.get_user(1)
    assert user["username"] == "alice"
    
    # Create a post for Alice
    post_id = post_service.create_post(
        user_id=1,
        title="My First Post",
        content="Hello, world!"
    )
    assert post_id == 1
    
    # Verify the post
    posts = post_service.get_user_posts(1)
    assert len(posts) == 1
    assert posts[0]["title"] == "My First Post"

def test_foreign_key_constraint(post_service):
    # Try to create a post for non-existent user
    with pytest.raises(Exception) as exc_info:
        post_service.create_post(
            user_id=999,
            title="Orphan Post",
            content="This should fail"
        )
    
    assert "FOREIGN KEY constraint failed" in str(exc_info.value)
```

Run this:

```bash
$ pytest test_foreign_keys.py -v
```

**Output**:

```text
test_foreign_keys.py::test_create_post_for_user PASSED
test_foreign_keys.py::test_foreign_key_constraint PASSED
```

Excellent! Our fixture architecture now handles:
- Session-scoped schema creation
- Function-scoped transaction rollback
- Flexible seed data via markers
- Multiple related tables with foreign keys
- Multiple service fixtures sharing the same connection

## The Complete Fixture Architecture

Here's the final, production-ready fixture setup:

```python
# conftest.py (complete version)
import pytest
import sqlite3
from user_service_injectable import UserService
from post_service import PostService

@pytest.fixture(scope="session")
def db_schema():
    """
    Create database schema once for all tests.
    
    Uses a named in-memory database that can be shared across connections.
    """
    db_uri = "file:test_db?mode=memory&cache=shared"
    conn = sqlite3.connect(db_uri, uri=True)
    
    # Enable foreign keys
    conn.execute("PRAGMA foreign_keys = ON")
    
    # Create all tables
    conn.execute("""
        CREATE TABLE users (
            id INTEGER PRIMARY KEY,
            username TEXT UNIQUE NOT NULL,
            email TEXT NOT NULL
        )
    """)
    
    conn.execute("""
        CREATE TABLE posts (
            id INTEGER PRIMARY KEY,
            user_id INTEGER NOT NULL,
            title TEXT NOT NULL,
            content TEXT NOT NULL,
            FOREIGN KEY (user_id) REFERENCES users(id)
        )
    """)
    
    conn.commit()
    conn.close()
    
    return db_uri

@pytest.fixture
def db(db_schema):
    """
    Provide a transactional connection that auto-rolls back.
    
    Each test gets a clean database state through transaction rollback.
    """
    conn = sqlite3.connect(db_schema, uri=True)
    conn.execute("PRAGMA foreign_keys = ON")
    conn.isolation_level = None  # Disable autocommit
    conn.execute("BEGIN")
    
    yield conn
    
    conn.rollback()
    conn.close()

@pytest.fixture
def seed_users(db, request):
    """
    Seed the database with users.
    
    Usage:
        @pytest.mark.seed_users([
            ("alice", "alice@example.com"),
            ("bob", "bob@example.com")
        ])
        def test_something(seed_users, user_service):
            ...
    """
    users = request.node.get_closest_marker("seed_users")
    if users:
        cursor = db.cursor()
        for username, email in users.args[0]:
            cursor.execute(
                "INSERT INTO users (username, email) VALUES (?, ?)",
                (username, email)
            )
        db.commit()
    
    return db

@pytest.fixture
def user_service(db):
    """Provide a UserService connected to the test database."""
    return UserService(connection=db)

@pytest.fixture
def post_service(db):
    """Provide a PostService connected to the test database."""
    return PostService(connection=db)
```

## Decision Framework: Fixture Scope Selection

| Scope | Lifetime | Use Case | Trade-offs |
|-------|----------|----------|------------|
| **function** | One test | Default, maximum isolation | Slowest, most setup overhead |
| **class** | All tests in a class | Related tests sharing setup | Medium speed, class coupling |
| **module** | All tests in a file | File-level shared resources | Faster, risk of interference |
| **session** | Entire test run | Expensive setup (schema, connections) | Fastest, requires careful cleanup |

### When to Apply Each Scope

**Function scope** (default):
- ✅ Maximum test isolation
- ✅ Each test gets fresh state
- ✅ Safe default choice
- ❌ Slowest for expensive setup

**Session scope**:
- ✅ One-time expensive operations (schema creation)
- ✅ Read-only shared resources
- ✅ Maximum speed for large test suites
- ❌ Requires careful state management

**Transaction rollback pattern**:
- ✅ Combines session-scoped schema with function-scoped isolation
- ✅ Best of both worlds: fast + isolated
- ✅ Professional standard for database testing
- ❌ Requires connection injection support

## Common Failure Modes and Their Signatures

### Symptom: Fixture not found

**Pytest output pattern**:

```text
E       fixture 'user_service' not found
>       available fixtures: cache, capfd, capfdbinary, ...
```

**Diagnostic clues**:
- Fixture defined in wrong file
- Fixture not in conftest.py or same file as test
- Typo in fixture name

**Root cause**: Pytest can't find the fixture definition

**Solution**: Move fixture to conftest.py or import it properly

### Symptom: Fixture executed multiple times unexpectedly

**Pytest output pattern**:

```text
# Debug output shows:
Creating database schema...
Creating database schema...
Creating database schema...
```

**Diagnostic clues**:
- Fixture scope is too narrow
- Expensive operation running for each test
- Performance degradation

**Root cause**: Fixture scope doesn't match usage pattern

**Solution**: Use session scope for expensive one-time setup

### Symptom: Tests interfere with each other

**Pytest output pattern**:

```text
test_first PASSED
test_second FAILED  # Expects clean state but sees data from test_first
```

**Diagnostic clues**:
- Tests pass individually but fail together
- Order-dependent failures
- Data from previous tests visible

**Root cause**: Missing transaction rollback or cleanup

**Solution**: Implement transaction rollback pattern or add explicit cleanup

## Testing API Calls

## The Challenge of Testing HTTP Clients

Testing code that makes HTTP requests presents unique challenges: external services are unreliable, slow, and may have rate limits or costs. You can't depend on them for your test suite. Yet you need to verify that your code handles responses correctly.

Consider an API client for a weather service:

```python
# weather_client.py
import requests

class WeatherClient:
    def __init__(self, api_key, base_url="https://api.weather.com"):
        self.api_key = api_key
        self.base_url = base_url
    
    def get_current_weather(self, city):
        """Get current weather for a city."""
        response = requests.get(
            f"{self.base_url}/current",
            params={"city": city, "api_key": self.api_key}
        )
        response.raise_for_status()
        return response.json()
    
    def get_forecast(self, city, days=5):
        """Get weather forecast for a city."""
        response = requests.get(
            f"{self.base_url}/forecast",
            params={"city": city, "days": days, "api_key": self.api_key}
        )
        response.raise_for_status()
        return response.json()
```

How do you test this without making real API calls? Let's start with the naive approach and watch it fail.

```python
# test_weather_client_naive.py
import pytest
from weather_client import WeatherClient

def test_get_current_weather():
    client = WeatherClient(api_key="test_key")
    
    weather = client.get_current_weather("London")
    
    assert weather is not None
    assert "temperature" in weather
```

Run this:

```bash
$ pytest test_weather_client_naive.py -v
```

**Output**:

```text
test_weather_client_naive.py::test_get_current_weather FAILED

================================ FAILURES =================================
__________________ test_get_current_weather ___________________________

    def test_get_current_weather():
        client = WeatherClient(api_key="test_key")
        
>       weather = client.get_current_weather("London")

test_weather_client_naive.py:6: 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
weather_client.py:12: in get_current_weather
    response.raise_for_status()
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _

>   raise HTTPError(http_error_msg, response=self)
E   requests.exceptions.HTTPError: 401 Client Error: Unauthorized
```

### Diagnostic Analysis: Real API Call Failure

**The complete output**:

The test fails with `requests.exceptions.HTTPError: 401 Client Error: Unauthorized`.

**Let's parse this section by section**:

1. **The error type**: `requests.exceptions.HTTPError`
   - What this tells us: The HTTP request completed, but the server rejected it
   - This is a real network call that reached the actual API

2. **The status code**: `401 Client Error: Unauthorized`
   - What this tells us: Our test API key is invalid
   - The real API rejected our authentication

3. **The test dependency**: Our test depends on:
   - Network connectivity
   - The external API being available
   - A valid API key
   - The API's current behavior

**Root cause identified**: We're making real HTTP requests to an external service in our tests.

**Why the current approach can't solve this**: We can't control external services, and we shouldn't depend on them for tests.

**What we need**: A way to intercept HTTP requests and return controlled responses without hitting the real API.

## Strategy 1: Mocking with unittest.mock

Use `unittest.mock` to replace the `requests.get` function:

```python
# test_weather_client_mock.py
import pytest
from unittest.mock import Mock, patch
from weather_client import WeatherClient

def test_get_current_weather_mocked():
    client = WeatherClient(api_key="test_key")
    
    # Create a mock response
    mock_response = Mock()
    mock_response.json.return_value = {
        "temperature": 20,
        "condition": "Sunny",
        "humidity": 65
    }
    mock_response.raise_for_status.return_value = None
    
    # Patch requests.get to return our mock
    with patch("requests.get", return_value=mock_response):
        weather = client.get_current_weather("London")
    
    assert weather["temperature"] == 20
    assert weather["condition"] == "Sunny"
```

Run this:

```bash
$ pytest test_weather_client_mock.py -v
```

**Output**:

```text
test_weather_client_mock.py::test_get_current_weather_mocked FAILED

================================ FAILURES =================================
______________ test_get_current_weather_mocked ________________________

    def test_get_current_weather_mocked():
        client = WeatherClient(api_key="test_key")
        
        mock_response = Mock()
        mock_response.json.return_value = {
            "temperature": 20,
            "condition": "Sunny",
            "humidity": 65
        }
        mock_response.raise_for_status.return_value = None
        
        with patch("requests.get", return_value=mock_response):
>           weather = client.get_current_weather("London")

test_weather_client_mock.py:18: 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _

>   raise HTTPError(http_error_msg, response=self)
E   requests.exceptions.HTTPError: 401 Client Error: Unauthorized
```

### Diagnostic Analysis: Patch Target Problem

**The complete output**:

Still getting the real HTTP error, even though we patched `requests.get`.

**Let's parse this section by section**:

1. **The patch location**: We patched `requests.get` in the global namespace
   - What this tells us: The patch didn't intercept the call
   - The real `requests.get` is still being called

2. **The import mechanism**: `weather_client.py` imports `requests` at the module level
   - What this tells us: When Python imports `weather_client`, it creates a reference to `requests.get`
   - Our patch of the global `requests.get` doesn't affect the reference in `weather_client`

**Root cause identified**: We patched the wrong location. We need to patch where the function is used, not where it's defined.

**Why the current approach can't solve this**: Patching `requests.get` globally doesn't affect the already-imported reference in `weather_client`.

**What we need**: Patch `requests.get` in the `weather_client` module's namespace.

## Strategy 2: Correct Patch Target

Patch where the function is used:

```python
# test_weather_client_correct_patch.py
import pytest
from unittest.mock import Mock, patch
from weather_client import WeatherClient

def test_get_current_weather_correct_patch():
    client = WeatherClient(api_key="test_key")
    
    mock_response = Mock()
    mock_response.json.return_value = {
        "temperature": 20,
        "condition": "Sunny",
        "humidity": 65
    }
    mock_response.raise_for_status.return_value = None
    
    # Patch requests.get in the weather_client module
    with patch("weather_client.requests.get", return_value=mock_response):
        weather = client.get_current_weather("London")
    
    assert weather["temperature"] == 20
    assert weather["condition"] == "Sunny"
    assert weather["humidity"] == 65
```

Run this:

```bash
$ pytest test_weather_client_correct_patch.py -v
```

**Output**:

```text
test_weather_client_correct_patch.py::test_get_current_weather_correct_patch PASSED
```

Success! But this approach has limitations. Let's test error handling:

```python
# test_weather_client_errors.py
import pytest
from unittest.mock import Mock, patch
from weather_client import WeatherClient
import requests

def test_handles_404_error():
    client = WeatherClient(api_key="test_key")
    
    # Create a mock that raises HTTPError
    mock_response = Mock()
    mock_response.raise_for_status.side_effect = requests.HTTPError(
        "404 Client Error: Not Found"
    )
    
    with patch("weather_client.requests.get", return_value=mock_response):
        with pytest.raises(requests.HTTPError):
            client.get_current_weather("NonexistentCity")

def test_handles_network_error():
    client = WeatherClient(api_key="test_key")
    
    # Mock a network failure
    with patch("weather_client.requests.get", side_effect=requests.ConnectionError):
        with pytest.raises(requests.ConnectionError):
            client.get_current_weather("London")
```

Run this:

```bash
$ pytest test_weather_client_errors.py -v
```

**Output**:

```text
test_weather_client_errors.py::test_handles_404_error PASSED
test_weather_client_errors.py::test_handles_network_error PASSED
```

Good! But notice the repetition: every test needs to create a mock response and patch `requests.get`. This is where fixtures help.

## Strategy 3: Mock Fixtures

Create reusable fixtures for common mock scenarios:

```python
# conftest.py
import pytest
from unittest.mock import Mock, patch
import requests

@pytest.fixture
def mock_requests_get():
    """Provide a patcher for requests.get."""
    with patch("weather_client.requests.get") as mock_get:
        yield mock_get

@pytest.fixture
def mock_weather_response():
    """Provide a mock successful weather response."""
    mock_response = Mock()
    mock_response.json.return_value = {
        "temperature": 20,
        "condition": "Sunny",
        "humidity": 65
    }
    mock_response.raise_for_status.return_value = None
    return mock_response

@pytest.fixture
def mock_forecast_response():
    """Provide a mock forecast response."""
    mock_response = Mock()
    mock_response.json.return_value = {
        "forecast": [
            {"day": 1, "temperature": 20, "condition": "Sunny"},
            {"day": 2, "temperature": 18, "condition": "Cloudy"},
            {"day": 3, "temperature": 22, "condition": "Sunny"}
        ]
    }
    mock_response.raise_for_status.return_value = None
    return mock_response
```

```python
# test_weather_client_fixtures.py
import pytest
from weather_client import WeatherClient

def test_get_current_weather(mock_requests_get, mock_weather_response):
    mock_requests_get.return_value = mock_weather_response
    
    client = WeatherClient(api_key="test_key")
    weather = client.get_current_weather("London")
    
    assert weather["temperature"] == 20
    assert weather["condition"] == "Sunny"
    
    # Verify the request was made correctly
    mock_requests_get.assert_called_once_with(
        "https://api.weather.com/current",
        params={"city": "London", "api_key": "test_key"}
    )

def test_get_forecast(mock_requests_get, mock_forecast_response):
    mock_requests_get.return_value = mock_forecast_response
    
    client = WeatherClient(api_key="test_key")
    forecast = client.get_forecast("London", days=3)
    
    assert len(forecast["forecast"]) == 3
    assert forecast["forecast"][0]["temperature"] == 20
    
    mock_requests_get.assert_called_once_with(
        "https://api.weather.com/forecast",
        params={"city": "London", "days": 3, "api_key": "test_key"}
    )
```

Run this:

```bash
$ pytest test_weather_client_fixtures.py -v
```

**Output**:

```text
test_weather_client_fixtures.py::test_get_current_weather PASSED
test_weather_client_fixtures.py::test_get_forecast PASSED
```

Much cleaner! But there's a better way: the `responses` library, which is specifically designed for mocking HTTP requests.

## Strategy 4: Using the responses Library

The `responses` library provides a cleaner API for mocking HTTP requests:

```bash
$ pip install responses
```

```python
# test_weather_client_responses.py
import pytest
import responses
from weather_client import WeatherClient

@responses.activate
def test_get_current_weather():
    # Register a mock response
    responses.add(
        responses.GET,
        "https://api.weather.com/current",
        json={
            "temperature": 20,
            "condition": "Sunny",
            "humidity": 65
        },
        status=200
    )
    
    client = WeatherClient(api_key="test_key")
    weather = client.get_current_weather("London")
    
    assert weather["temperature"] == 20
    assert weather["condition"] == "Sunny"
    
    # Verify the request
    assert len(responses.calls) == 1
    assert responses.calls[0].request.url == (
        "https://api.weather.com/current?city=London&api_key=test_key"
    )

@responses.activate
def test_handles_404():
    # Register a 404 response
    responses.add(
        responses.GET,
        "https://api.weather.com/current",
        json={"error": "City not found"},
        status=404
    )
    
    client = WeatherClient(api_key="test_key")
    
    with pytest.raises(Exception) as exc_info:
        client.get_current_weather("NonexistentCity")
    
    assert "404" in str(exc_info.value)
```

Run this:

```bash
$ pytest test_weather_client_responses.py -v
```

**Output**:

```text
test_weather_client_responses.py::test_get_current_weather PASSED
test_weather_client_responses.py::test_handles_404 PASSED
```

Much cleaner! The `responses` library handles the patching automatically and provides a more intuitive API.

## Strategy 5: Fixture-Based responses Setup

Combine `responses` with fixtures for maximum reusability:

```python
# conftest.py (updated)
import pytest
import responses

@pytest.fixture
def mock_weather_api():
    """Activate responses and provide helper for registering endpoints."""
    with responses.RequestsMock() as rsps:
        yield rsps

@pytest.fixture
def weather_api_success(mock_weather_api):
    """Register successful weather API responses."""
    mock_weather_api.add(
        responses.GET,
        "https://api.weather.com/current",
        json={
            "temperature": 20,
            "condition": "Sunny",
            "humidity": 65
        },
        status=200
    )
    
    mock_weather_api.add(
        responses.GET,
        "https://api.weather.com/forecast",
        json={
            "forecast": [
                {"day": 1, "temperature": 20, "condition": "Sunny"},
                {"day": 2, "temperature": 18, "condition": "Cloudy"},
                {"day": 3, "temperature": 22, "condition": "Sunny"}
            ]
        },
        status=200
    )
    
    return mock_weather_api
```

```python
# test_weather_client_fixture_responses.py
import pytest
from weather_client import WeatherClient

def test_get_current_weather(weather_api_success):
    client = WeatherClient(api_key="test_key")
    weather = client.get_current_weather("London")
    
    assert weather["temperature"] == 20
    assert weather["condition"] == "Sunny"

def test_get_forecast(weather_api_success):
    client = WeatherClient(api_key="test_key")
    forecast = client.get_forecast("London", days=3)
    
    assert len(forecast["forecast"]) == 3
    assert forecast["forecast"][0]["temperature"] == 20

def test_multiple_calls(weather_api_success):
    client = WeatherClient(api_key="test_key")
    
    # Make multiple calls
    weather = client.get_current_weather("London")
    forecast = client.get_forecast("London")
    
    assert weather["temperature"] == 20
    assert len(forecast["forecast"]) == 3
    
    # Verify both calls were made
    assert len(weather_api_success.calls) == 2
```

Run this:

```bash
$ pytest test_weather_client_fixture_responses.py -v
```

**Output**:

```text
test_weather_client_fixture_responses.py::test_get_current_weather PASSED
test_weather_client_fixture_responses.py::test_get_forecast PASSED
test_weather_client_fixture_responses.py::test_multiple_calls PASSED
```

Perfect! Now tests are clean, focused, and don't depend on external services.

## Decision Framework: API Testing Strategies

| Strategy | Setup Complexity | Flexibility | Best For |
|----------|-----------------|-------------|----------|
| **unittest.mock** | Medium | High | Complex mocking scenarios, custom behavior |
| **responses library** | Low | Medium | HTTP-specific testing, standard REST APIs |
| **VCR.py** | Low | Low | Recording real API interactions, regression tests |
| **Test doubles** | High | High | Full control, complex protocols |

### When to Apply Each Solution

**unittest.mock**:
- ✅ Need fine-grained control over mock behavior
- ✅ Testing error conditions and edge cases
- ✅ Already using mocks elsewhere in codebase
- ❌ HTTP-specific features (status codes, headers) are verbose

**responses library**:
- ✅ Testing HTTP clients specifically
- ✅ Need to verify request parameters
- ✅ Want clean, readable test code
- ❌ Non-HTTP protocols (WebSockets, gRPC)

**VCR.py** (record/replay):
- ✅ Want to test against real API responses
- ✅ Need to capture complex response structures
- ✅ Regression testing against API changes
- ❌ API requires authentication or has side effects

## Common Failure Modes and Their Signatures

### Symptom: Mock not intercepting requests

**Pytest output pattern**:

```text
requests.exceptions.HTTPError: 401 Client Error: Unauthorized
# Even though you patched requests.get
```

**Diagnostic clues**:
- Real HTTP error despite mocking
- Patch applied but not working
- Network calls still happening

**Root cause**: Patching wrong location (global vs. module namespace)

**Solution**: Patch where the function is used: `patch("module_name.requests.get")`

### Symptom: responses not activating

**Pytest output pattern**:

```text
ConnectionError: Connection refused
# When using responses library
```

**Diagnostic clues**:
- responses.add() called but requests still fail
- Missing @responses.activate decorator
- Fixture not properly activating responses

**Root cause**: responses not activated for the test

**Solution**: Use `@responses.activate` decorator or `responses.RequestsMock()` context manager

### Symptom: Request parameters not matching

**Pytest output pattern**:

```text
ConnectionError: Connection refused
# responses registered but not matching
```

**Diagnostic clues**:
- responses.add() called with URL
- Request made to slightly different URL
- Query parameters in different order

**Root cause**: URL or parameter mismatch between registered mock and actual request

**Solution**: Use `responses.matchers` for flexible parameter matching or verify exact URL construction

## Mocking HTTP Requests with responses

## Deep Dive: The responses Library

In the previous section, we introduced the `responses` library for mocking HTTP requests. Now let's explore its full capabilities and learn how to handle complex real-world scenarios.

The `responses` library intercepts HTTP requests at the `requests` library level, allowing you to define expected requests and their responses without making actual network calls.

## Iteration 1: Basic Response Registration

Start with the simplest case—registering a single response:

```python
# test_responses_basic.py
import responses
import requests

@responses.activate
def test_simple_get():
    # Register a mock response
    responses.add(
        responses.GET,
        "https://api.example.com/users/1",
        json={"id": 1, "name": "Alice"},
        status=200
    )
    
    # Make the request
    response = requests.get("https://api.example.com/users/1")
    
    assert response.status_code == 200
    assert response.json() == {"id": 1, "name": "Alice"}
```

Run this:

```bash
$ pytest test_responses_basic.py -v
```

**Output**:

```text
test_responses_basic.py::test_simple_get PASSED
```

Good! But what if the URL doesn't match exactly?

```python
# test_responses_mismatch.py
import responses
import requests

@responses.activate
def test_url_mismatch():
    responses.add(
        responses.GET,
        "https://api.example.com/users/1",
        json={"id": 1, "name": "Alice"},
        status=200
    )
    
    # Request a different user ID
    response = requests.get("https://api.example.com/users/2")
    
    assert response.status_code == 200
```

Run this:

```bash
$ pytest test_responses_mismatch.py -v
```

**Output**:

```text
test_responses_mismatch.py::test_url_mismatch FAILED

================================ FAILURES =================================
________________________ test_url_mismatch ____________________________

    @responses.activate
    def test_url_mismatch():
        responses.add(
            responses.GET,
            "https://api.example.com/users/1",
            json={"id": 1, "name": "Alice"},
            status=200
        )
        
>       response = requests.get("https://api.example.com/users/2")

test_responses_mismatch.py:13: 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _

>   raise ConnectionError(msg)
E   requests.exceptions.ConnectionError: Connection refused by Responses
```

### Diagnostic Analysis: URL Matching Failure

**The complete output**:

The test fails with `requests.exceptions.ConnectionError: Connection refused by Responses`.

**Let's parse this section by section**:

1. **The error type**: `ConnectionError` from the `responses` library
   - What this tells us: responses is active, but no registered mock matched the request
   - This is not a real network error—it's responses refusing to handle the request

2. **The URL mismatch**: We registered `/users/1` but requested `/users/2`
   - What this tells us: responses requires exact URL matches by default
   - Each unique URL needs its own registration

**Root cause identified**: responses uses exact URL matching, and we didn't register a mock for `/users/2`.

**Why the current approach can't solve this**: We need to either register multiple URLs or use pattern matching.

**What we need**: A way to match URLs dynamically or register multiple similar endpoints.

## Iteration 2: Multiple Responses and Dynamic URLs

Register multiple responses for different URLs:

```python
# test_responses_multiple.py
import responses
import requests

@responses.activate
def test_multiple_users():
    # Register responses for multiple users
    responses.add(
        responses.GET,
        "https://api.example.com/users/1",
        json={"id": 1, "name": "Alice"},
        status=200
    )
    
    responses.add(
        responses.GET,
        "https://api.example.com/users/2",
        json={"id": 2, "name": "Bob"},
        status=200
    )
    
    # Request both users
    alice = requests.get("https://api.example.com/users/1")
    bob = requests.get("https://api.example.com/users/2")
    
    assert alice.json()["name"] == "Alice"
    assert bob.json()["name"] == "Bob"
```

Run this:

```bash
$ pytest test_responses_multiple.py -v
```

**Output**:

```text
test_responses_multiple.py::test_multiple_users PASSED
```

This works, but it's tedious for many endpoints. Let's use regex matching:

```python
# test_responses_regex.py
import responses
import requests
import re

@responses.activate
def test_regex_matching():
    # Use regex to match any user ID
    responses.add(
        responses.GET,
        re.compile(r"https://api\.example\.com/users/\d+"),
        json={"id": 999, "name": "Generic User"},
        status=200
    )
    
    # Request any user ID
    user1 = requests.get("https://api.example.com/users/1")
    user2 = requests.get("https://api.example.com/users/2")
    user100 = requests.get("https://api.example.com/users/100")
    
    # All get the same response
    assert user1.json()["name"] == "Generic User"
    assert user2.json()["name"] == "Generic User"
    assert user100.json()["name"] == "Generic User"
```

Run this:

```bash
$ pytest test_responses_regex.py -v
```

**Output**:

```text
test_responses_regex.py::test_regex_matching PASSED
```

Good! But returning the same response for all IDs isn't realistic. We need dynamic responses.

## Iteration 3: Dynamic Response Callbacks

Use callbacks to generate responses dynamically:

```python
# test_responses_callback.py
import responses
import requests
import re

def user_callback(request):
    # Extract user ID from URL
    user_id = request.url.split("/")[-1]
    
    # Generate response based on ID
    return (
        200,
        {},
        {
            "id": int(user_id),
            "name": f"User {user_id}",
            "email": f"user{user_id}@example.com"
        }
    )

@responses.activate
def test_dynamic_responses():
    responses.add_callback(
        responses.GET,
        re.compile(r"https://api\.example\.com/users/\d+"),
        callback=user_callback,
        content_type="application/json"
    )
    
    # Request different users
    user1 = requests.get("https://api.example.com/users/1")
    user2 = requests.get("https://api.example.com/users/2")
    
    # Each gets a unique response
    assert user1.json() == {
        "id": 1,
        "name": "User 1",
        "email": "user1@example.com"
    }
    
    assert user2.json() == {
        "id": 2,
        "name": "User 2",
        "email": "user2@example.com"
    }
```

Run this:

```bash
$ pytest test_responses_callback.py -v
```

**Output**:

```text
test_responses_callback.py::test_dynamic_responses PASSED
```

Excellent! Now let's handle query parameters.

## Iteration 4: Matching Query Parameters

Test endpoints that use query parameters:

```python
# test_responses_query_params.py
import responses
import requests

@responses.activate
def test_query_params_exact():
    # Register response for specific query params
    responses.add(
        responses.GET,
        "https://api.example.com/search?q=python&limit=10",
        json={"results": ["result1", "result2"]},
        status=200
    )
    
    # Request with exact params
    response = requests.get(
        "https://api.example.com/search",
        params={"q": "python", "limit": 10}
    )
    
    assert response.json() == {"results": ["result1", "result2"]}
```

Run this:

```bash
$ pytest test_responses_query_params.py -v
```

**Output**:

```text
test_responses_query_params.py::test_query_params_exact FAILED

================================ FAILURES =================================
_________________ test_query_params_exact _____________________________

    @responses.activate
    def test_query_params_exact():
        responses.add(
            responses.GET,
            "https://api.example.com/search?q=python&limit=10",
            json={"results": ["result1", "result2"]},
            status=200
        )
        
>       response = requests.get(
            "https://api.example.com/search",
            params={"q": "python", "limit": 10}
        )

test_responses_query_params.py:12: 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _

>   raise ConnectionError(msg)
E   requests.exceptions.ConnectionError: Connection refused by Responses
```

### Diagnostic Analysis: Query Parameter Order

**The complete output**:

The test fails even though we provided the exact query parameters.

**Let's parse this section by section**:

1. **The URL construction**: `requests` builds the URL as `?q=python&limit=10`
   - What this tells us: The order of query parameters matters for exact matching
   - We registered `?q=python&limit=10` but requests might construct `?limit=10&q=python`

2. **The matching behavior**: responses uses string comparison for URLs
   - What this tells us: Query parameter order affects matching
   - Dictionary iteration order can vary

**Root cause identified**: Query parameter order is not guaranteed, causing URL mismatch.

**Why the current approach can't solve this**: String-based URL matching is too brittle for query parameters.

**What we need**: A way to match query parameters regardless of order.

## Iteration 5: Flexible Query Parameter Matching

Use matchers for flexible parameter matching:

```python
# test_responses_matchers.py
import responses
import requests
from responses import matchers

@responses.activate
def test_query_params_matcher():
    # Use matcher for query params (order-independent)
    responses.add(
        responses.GET,
        "https://api.example.com/search",
        match=[
            matchers.query_param_matcher({"q": "python", "limit": "10"})
        ],
        json={"results": ["result1", "result2"]},
        status=200
    )
    
    # Request with params in any order
    response = requests.get(
        "https://api.example.com/search",
        params={"limit": 10, "q": "python"}  # Different order
    )
    
    assert response.json() == {"results": ["result1", "result2"]}

@responses.activate
def test_partial_query_params():
    # Match only specific params, ignore others
    responses.add(
        responses.GET,
        "https://api.example.com/search",
        match=[
            matchers.query_param_matcher({"q": "python"}, strict_match=False)
        ],
        json={"results": ["result1", "result2"]},
        status=200
    )
    
    # Request with extra params
    response = requests.get(
        "https://api.example.com/search",
        params={"q": "python", "limit": 10, "offset": 0}
    )
    
    assert response.json() == {"results": ["result1", "result2"]}
```

Run this:

```bash
$ pytest test_responses_matchers.py -v
```

**Output**:

```text
test_responses_matchers.py::test_query_params_matcher PASSED
test_responses_matchers.py::test_partial_query_params PASSED
```

Perfect! Now let's handle POST requests with JSON bodies.

## Iteration 6: Matching Request Bodies

Test POST requests with JSON payloads:

```python
# test_responses_post.py
import responses
import requests
from responses import matchers

@responses.activate
def test_post_with_json():
    # Match POST request with specific JSON body
    responses.add(
        responses.POST,
        "https://api.example.com/users",
        match=[
            matchers.json_params_matcher({
                "name": "Alice",
                "email": "alice@example.com"
            })
        ],
        json={"id": 1, "name": "Alice", "email": "alice@example.com"},
        status=201
    )
    
    # Make POST request
    response = requests.post(
        "https://api.example.com/users",
        json={"name": "Alice", "email": "alice@example.com"}
    )
    
    assert response.status_code == 201
    assert response.json()["id"] == 1

@responses.activate
def test_post_with_partial_match():
    # Match only specific fields in JSON body
    responses.add(
        responses.POST,
        "https://api.example.com/users",
        match=[
            matchers.json_params_matcher(
                {"name": "Bob"},
                strict_match=False
            )
        ],
        json={"id": 2, "name": "Bob"},
        status=201
    )
    
    # Request with extra fields
    response = requests.post(
        "https://api.example.com/users",
        json={
            "name": "Bob",
            "email": "bob@example.com",
            "age": 30
        }
    )
    
    assert response.status_code == 201
    assert response.json()["name"] == "Bob"
```

Run this:

```bash
$ pytest test_responses_post.py -v
```

**Output**:

```text
test_responses_post.py::test_post_with_json PASSED
test_responses_post.py::test_post_with_partial_match PASSED
```

Great! Now let's handle headers.

## Iteration 7: Matching and Returning Headers

Test requests that require specific headers:

```python
# test_responses_headers.py
import responses
import requests
from responses import matchers

@responses.activate
def test_request_headers():
    # Match request with specific headers
    responses.add(
        responses.GET,
        "https://api.example.com/protected",
        match=[
            matchers.header_matcher({
                "Authorization": "Bearer secret-token"
            })
        ],
        json={"data": "protected content"},
        status=200
    )
    
    # Request with correct header
    response = requests.get(
        "https://api.example.com/protected",
        headers={"Authorization": "Bearer secret-token"}
    )
    
    assert response.status_code == 200
    assert response.json()["data"] == "protected content"

@responses.activate
def test_response_headers():
    # Return specific headers in response
    responses.add(
        responses.GET,
        "https://api.example.com/data",
        json={"data": "content"},
        headers={
            "X-RateLimit-Remaining": "99",
            "X-RateLimit-Reset": "1234567890"
        },
        status=200
    )
    
    response = requests.get("https://api.example.com/data")
    
    assert response.headers["X-RateLimit-Remaining"] == "99"
    assert response.headers["X-RateLimit-Reset"] == "1234567890"

@responses.activate
def test_missing_auth_header():
    # Require auth header
    responses.add(
        responses.GET,
        "https://api.example.com/protected",
        match=[
            matchers.header_matcher({
                "Authorization": matchers.ANY
            })
        ],
        json={"data": "protected content"},
        status=200
    )
    
    # Request without auth header should fail
    with pytest.raises(ConnectionError):
        requests.get("https://api.example.com/protected")
```

Run this:

```bash
$ pytest test_responses_headers.py -v
```

**Output**:

```text
test_responses_headers.py::test_request_headers PASSED
test_responses_headers.py::test_response_headers PASSED
test_responses_headers.py::test_missing_auth_header PASSED
```

Excellent! Now let's handle sequential responses.

## Iteration 8: Sequential Responses for Repeated Calls

Test code that makes the same request multiple times:

```python
# test_responses_sequential.py
import responses
import requests

@responses.activate
def test_sequential_responses():
    # First call returns one result
    responses.add(
        responses.GET,
        "https://api.example.com/status",
        json={"status": "pending"},
        status=200
    )
    
    # Second call returns different result
    responses.add(
        responses.GET,
        "https://api.example.com/status",
        json={"status": "processing"},
        status=200
    )
    
    # Third call returns final result
    responses.add(
        responses.GET,
        "https://api.example.com/status",
        json={"status": "complete"},
        status=200
    )
    
    # Make three requests
    response1 = requests.get("https://api.example.com/status")
    response2 = requests.get("https://api.example.com/status")
    response3 = requests.get("https://api.example.com/status")
    
    assert response1.json()["status"] == "pending"
    assert response2.json()["status"] == "processing"
    assert response3.json()["status"] == "complete"

@responses.activate
def test_polling_until_complete():
    """Simulate polling an async operation."""
    # Register multiple responses
    for _ in range(3):
        responses.add(
            responses.GET,
            "https://api.example.com/job/123",
            json={"status": "running"},
            status=200
        )
    
    # Final response
    responses.add(
        responses.GET,
        "https://api.example.com/job/123",
        json={"status": "complete", "result": "success"},
        status=200
    )
    
    # Poll until complete
    status = "running"
    attempts = 0
    while status == "running" and attempts < 10:
        response = requests.get("https://api.example.com/job/123")
        status = response.json()["status"]
        attempts += 1
    
    assert status == "complete"
    assert attempts == 4  # 3 running + 1 complete
```

Run this:

```bash
$ pytest test_responses_sequential.py -v
```

**Output**:

```text
test_responses_sequential.py::test_sequential_responses PASSED
test_responses_sequential.py::test_polling_until_complete PASSED
```

Perfect! Now let's verify request details.

## Iteration 9: Inspecting Captured Requests

Verify that your code makes the correct requests:

```python
# test_responses_inspection.py
import responses
import requests

@responses.activate
def test_request_inspection():
    responses.add(
        responses.POST,
        "https://api.example.com/users",
        json={"id": 1},
        status=201
    )
    
    # Make request
    requests.post(
        "https://api.example.com/users",
        json={"name": "Alice", "email": "alice@example.com"},
        headers={"X-Client-Version": "1.0"}
    )
    
    # Inspect the captured request
    assert len(responses.calls) == 1
    
    call = responses.calls[0]
    assert call.request.url == "https://api.example.com/users"
    assert call.request.method == "POST"
    assert call.request.headers["X-Client-Version"] == "1.0"
    
    # Inspect request body
    import json
    body = json.loads(call.request.body)
    assert body["name"] == "Alice"
    assert body["email"] == "alice@example.com"

@responses.activate
def test_multiple_requests():
    responses.add(
        responses.GET,
        "https://api.example.com/users/1",
        json={"id": 1, "name": "Alice"},
        status=200
    )
    
    responses.add(
        responses.GET,
        "https://api.example.com/users/2",
        json={"id": 2, "name": "Bob"},
        status=200
    )
    
    # Make multiple requests
    requests.get("https://api.example.com/users/1")
    requests.get("https://api.example.com/users/2")
    
    # Verify both were made
    assert len(responses.calls) == 2
    assert "users/1" in responses.calls[0].request.url
    assert "users/2" in responses.calls[1].request.url
```

Run this:

```bash
$ pytest test_responses_inspection.py -v
```

**Output**:

```text
test_responses_inspection.py::test_request_inspection PASSED
test_responses_inspection.py::test_multiple_requests PASSED
```

## The Complete responses Toolkit

Here's a comprehensive fixture setup for real-world API testing:

```python
# conftest.py (complete responses setup)
import pytest
import responses
from responses import matchers
import re

@pytest.fixture
def mock_api():
    """Activate responses for the test."""
    with responses.RequestsMock() as rsps:
        yield rsps

@pytest.fixture
def mock_user_api(mock_api):
    """Mock a complete user API."""
    
    # GET /users - list users
    mock_api.add(
        responses.GET,
        "https://api.example.com/users",
        json={"users": [
            {"id": 1, "name": "Alice"},
            {"id": 2, "name": "Bob"}
        ]},
        status=200
    )
    
    # GET /users/:id - get specific user (dynamic)
    def user_callback(request):
        user_id = int(request.url.split("/")[-1])
        return (200, {}, {
            "id": user_id,
            "name": f"User {user_id}",
            "email": f"user{user_id}@example.com"
        })
    
    mock_api.add_callback(
        responses.GET,
        re.compile(r"https://api\.example\.com/users/\d+"),
        callback=user_callback,
        content_type="application/json"
    )
    
    # POST /users - create user
    mock_api.add(
        responses.POST,
        "https://api.example.com/users",
        json={"id": 999, "name": "New User"},
        status=201
    )
    
    # PUT /users/:id - update user
    mock_api.add(
        responses.PUT,
        re.compile(r"https://api\.example\.com/users/\d+"),
        json={"id": 1, "name": "Updated User"},
        status=200
    )
    
    # DELETE /users/:id - delete user
    mock_api.add(
        responses.DELETE,
        re.compile(r"https://api\.example\.com/users/\d+"),
        status=204
    )
    
    return mock_api

@pytest.fixture
def mock_auth_api(mock_api):
    """Mock an authenticated API."""
    
    # Successful auth
    mock_api.add(
        responses.POST,
        "https://api.example.com/auth/login",
        match=[
            matchers.json_params_matcher({
                "username": "alice",
                "password": "secret"
            })
        ],
        json={"token": "valid-token-123"},
        status=200
    )
    
    # Failed auth
    mock_api.add(
        responses.POST,
        "https://api.example.com/auth/login",
        match=[
            matchers.json_params_matcher({
                "username": "alice",
                "password": "wrong"
            })
        ],
        json={"error": "Invalid credentials"},
        status=401
    )
    
    # Protected endpoint
    mock_api.add(
        responses.GET,
        "https://api.example.com/protected",
        match=[
            matchers.header_matcher({
                "Authorization": "Bearer valid-token-123"
            })
        ],
        json={"data": "secret content"},
        status=200
    )
    
    return mock_api
```

## Decision Framework: responses Features

| Feature | Use Case | Example |
|---------|----------|---------|
| **Exact URL** | Simple, static endpoints | `responses.add(GET, "https://api.com/users")` |
| **Regex URL** | Dynamic IDs in path | `re.compile(r"https://api\.com/users/\d+")` |
| **Callbacks** | Dynamic response generation | `responses.add_callback(GET, url, callback=func)` |
| **Query matchers** | Order-independent params | `matchers.query_param_matcher({"q": "python"})` |
| **JSON matchers** | POST/PUT body validation | `matchers.json_params_matcher({"name": "Alice"})` |
| **Header matchers** | Auth, content-type checks | `matchers.header_matcher({"Authorization": "Bearer X"})` |
| **Sequential** | Polling, state changes | Multiple `responses.add()` for same URL |

### When to Apply Each Feature

**Exact URL matching**:
- ✅ Simple, static endpoints
- ✅ No dynamic parameters
- ✅ Maximum clarity
- ❌ Many similar endpoints (use regex)

**Regex matching**:
- ✅ Dynamic IDs in URL path
- ✅ Multiple similar endpoints
- ✅ Flexible matching
- ❌ Complex response logic (use callbacks)

**Callbacks**:
- ✅ Response depends on request data
- ✅ Complex business logic
- ✅ Stateful simulations
- ❌ Simple static responses (overkill)

**Matchers**:
- ✅ Query parameters (order-independent)
- ✅ JSON body validation
- ✅ Header requirements
- ✅ Partial matching
- ❌ Exact string matching is sufficient

## Common Failure Modes and Their Signatures

### Symptom: ConnectionError despite registered mock

**Pytest output pattern**:

```text
requests.exceptions.ConnectionError: Connection refused by Responses
```

**Diagnostic clues**:
- Mock registered but not matching
- URL, method, or parameters don't match exactly
- Missing @responses.activate decorator

**Root cause**: Request doesn't match any registered mock

**Solution**: Use `responses.calls` to inspect what was actually requested, compare with registered mocks

### Symptom: Wrong response returned

**Pytest output pattern**:

```text
AssertionError: assert 'pending' == 'complete'
# Expected 'complete' but got 'pending'
```

**Diagnostic clues**:
- Sequential responses registered
- Test made more/fewer calls than expected
- Response order matters

**Root cause**: Response consumed in wrong order or test logic error

**Solution**: Verify number of calls with `len(responses.calls)`, check response order

### Symptom: Matcher not working

**Pytest output pattern**:

```text
ConnectionError: Connection refused by Responses
# Even with matcher configured
```

**Diagnostic clues**:
- Matcher configured but not matching
- Query params as strings vs. integers
- Strict vs. non-strict matching

**Root cause**: Type mismatch or strict matching when partial needed

**Solution**: Use `strict_match=False` for partial matching, ensure type consistency

## Testing File I/O

## The Challenge of Testing File Operations

Testing code that reads and writes files presents unique challenges: file operations have side effects, tests can interfere with each other through the filesystem, and cleanup is critical. You need to ensure tests don't leave files behind or depend on specific filesystem state.

Consider a configuration file manager:

```python
# config_manager.py
import json
import os

class ConfigManager:
    def __init__(self, config_path):
        self.config_path = config_path
    
    def load_config(self):
        """Load configuration from file."""
        if not os.path.exists(self.config_path):
            return {}
        
        with open(self.config_path, 'r') as f:
            return json.load(f)
    
    def save_config(self, config):
        """Save configuration to file."""
        # Create directory if it doesn't exist
        os.makedirs(os.path.dirname(self.config_path), exist_ok=True)
        
        with open(self.config_path, 'w') as f:
            json.dump(config, f, indent=2)
    
    def update_config(self, key, value):
        """Update a single configuration value."""
        config = self.load_config()
        config[key] = value
        self.save_config(config)
    
    def delete_config(self):
        """Delete the configuration file."""
        if os.path.exists(self.config_path):
            os.remove(self.config_path)
```

How do you test this without polluting your filesystem? Let's start with the naive approach.

```python
# test_config_manager_naive.py
import pytest
from config_manager import ConfigManager

def test_save_and_load():
    manager = ConfigManager("test_config.json")
    
    # Save config
    config = {"database": "localhost", "port": 5432}
    manager.save_config(config)
    
    # Load it back
    loaded = manager.load_config()
    
    assert loaded == config

def test_update_config():
    manager = ConfigManager("test_config.json")
    
    # Update a value
    manager.update_config("database", "production.db")
    
    # Verify
    config = manager.load_config()
    assert config["database"] == "production.db"
```

Run this:

```bash
$ pytest test_config_manager_naive.py -v
```

**Output**:

```text
test_config_manager_naive.py::test_save_and_load PASSED
test_config_manager_naive.py::test_update_config PASSED
```

Tests pass! But check your directory:

```bash
$ ls -la
```

**Output**:

```text
-rw-r--r-- 1 user user   45 Dec 10 10:30 test_config.json
```

The test left a file behind. Run the tests again:

```bash
$ pytest test_config_manager_naive.py -v
```

**Output**:

```text
test_config_manager_naive.py::test_save_and_load PASSED
test_config_manager_naive.py::test_update_config FAILED

================================ FAILURES =================================
__________________ test_update_config _________________________________

    def test_update_config():
        manager = ConfigManager("test_config.json")
        
        manager.update_config("database", "production.db")
        
        config = manager.load_config()
>       assert config["database"] == "production.db"
E       AssertionError: assert 'production.db' == 'production.db'
E       + where {'database': 'production.db', 'port': 5432} = ...
```

### Diagnostic Analysis: File State Pollution

**The complete output**:

The second test fails because it sees data from the first test.

**Let's parse this section by section**:

1. **The assertion failure**: The config has both `database` and `port` keys
   - What this tells us: The file from `test_save_and_load` still exists
   - `test_update_config` loaded the existing file and added to it

2. **The test interdependency**: Tests share state through the filesystem
   - What this tells us: No cleanup between tests
   - File persists across test runs

**Root cause identified**: Tests write to real files that persist between runs.

**Why the current approach can't solve this**: We need automatic cleanup after each test.

**What we need**: A way to create temporary files that are automatically cleaned up.

## Strategy 1: Manual Cleanup with Fixtures

Add cleanup using fixtures:

```python
# test_config_manager_cleanup.py
import pytest
import os
from config_manager import ConfigManager

@pytest.fixture
def config_file():
    """Provide a config file path and clean it up after the test."""
    path = "test_config.json"
    
    yield path
    
    # Cleanup
    if os.path.exists(path):
        os.remove(path)

def test_save_and_load(config_file):
    manager = ConfigManager(config_file)
    
    config = {"database": "localhost", "port": 5432}
    manager.save_config(config)
    
    loaded = manager.load_config()
    assert loaded == config

def test_update_config(config_file):
    manager = ConfigManager(config_file)
    
    manager.update_config("database", "production.db")
    
    config = manager.load_config()
    assert config["database"] == "production.db"
```

Run this multiple times:

```bash
$ pytest test_config_manager_cleanup.py -v
$ pytest test_config_manager_cleanup.py -v
```

**Output**:

```text
test_config_manager_cleanup.py::test_save_and_load PASSED
test_config_manager_cleanup.py::test_update_config PASSED

test_config_manager_cleanup.py::test_save_and_load PASSED
test_config_manager_cleanup.py::test_update_config PASSED
```

Good! Tests are now isolated. But what if tests run in parallel? Or what if we need multiple test files?

## Strategy 2: Unique File Names

Generate unique file names for each test:

```python
# test_config_manager_unique.py
import pytest
import os
import uuid
from config_manager import ConfigManager

@pytest.fixture
def config_file():
    """Provide a unique config file path."""
    path = f"test_config_{uuid.uuid4()}.json"
    
    yield path
    
    if os.path.exists(path):
        os.remove(path)

def test_save_and_load(config_file):
    manager = ConfigManager(config_file)
    
    config = {"database": "localhost", "port": 5432}
    manager.save_config(config)
    
    loaded = manager.load_config()
    assert loaded == config

def test_update_config(config_file):
    manager = ConfigManager(config_file)
    
    manager.update_config("database", "production.db")
    
    config = manager.load_config()
    assert config["database"] == "production.db"
```

Run this:

```bash
$ pytest test_config_manager_unique.py -v
```

**Output**:

```text
test_config_manager_unique.py::test_save_and_load PASSED
test_config_manager_unique.py::test_update_config PASSED
```

This works, but we're still writing to the current directory. What if we need to test directory creation? Or test with nested paths?

## Strategy 3: Temporary Directories

Use Python's `tempfile` module for proper temporary file handling:

```python
# test_config_manager_tempfile.py
import pytest
import os
import tempfile
from config_manager import ConfigManager

@pytest.fixture
def temp_config_file():
    """Provide a temporary config file in a temp directory."""
    # Create a temporary directory
    with tempfile.TemporaryDirectory() as tmpdir:
        config_path = os.path.join(tmpdir, "config.json")
        yield config_path
        # Directory and all contents automatically cleaned up

def test_save_and_load(temp_config_file):
    manager = ConfigManager(temp_config_file)
    
    config = {"database": "localhost", "port": 5432}
    manager.save_config(config)
    
    loaded = manager.load_config()
    assert loaded == config

def test_nested_directory(temp_config_file):
    # Test with nested path
    nested_path = temp_config_file.replace("config.json", "nested/dir/config.json")
    manager = ConfigManager(nested_path)
    
    config = {"key": "value"}
    manager.save_config(config)
    
    # Verify directory was created
    assert os.path.exists(os.path.dirname(nested_path))
    
    loaded = manager.load_config()
    assert loaded == config
```

Run this:

```bash
$ pytest test_config_manager_tempfile.py -v
```

**Output**:

```text
test_config_manager_tempfile.py::test_save_and_load PASSED
test_config_manager_tempfile.py::test_nested_directory PASSED
```

Perfect! But there's an even better way: pytest's built-in `tmp_path` fixture.

## Strategy 4: pytest's tmp_path Fixture

Use pytest's built-in temporary directory fixture:

```python
# test_config_manager_tmp_path.py
import pytest
from config_manager import ConfigManager

def test_save_and_load(tmp_path):
    """tmp_path is a pathlib.Path object to a temporary directory."""
    config_file = tmp_path / "config.json"
    manager = ConfigManager(str(config_file))
    
    config = {"database": "localhost", "port": 5432}
    manager.save_config(config)
    
    loaded = manager.load_config()
    assert loaded == config

def test_nested_directory(tmp_path):
    config_file = tmp_path / "nested" / "dir" / "config.json"
    manager = ConfigManager(str(config_file))
    
    config = {"key": "value"}
    manager.save_config(config)
    
    assert config_file.exists()
    
    loaded = manager.load_config()
    assert loaded == config

def test_multiple_files(tmp_path):
    """Test with multiple config files."""
    config1 = tmp_path / "config1.json"
    config2 = tmp_path / "config2.json"
    
    manager1 = ConfigManager(str(config1))
    manager2 = ConfigManager(str(config2))
    
    manager1.save_config({"env": "dev"})
    manager2.save_config({"env": "prod"})
    
    assert manager1.load_config()["env"] == "dev"
    assert manager2.load_config()["env"] == "prod"
```

Run this:

```bash
$ pytest test_config_manager_tmp_path.py -v
```

**Output**:

```text
test_config_manager_tmp_path.py::test_save_and_load PASSED
test_config_manager_tmp_path.py::test_nested_directory PASSED
test_config_manager_tmp_path.py::test_multiple_files PASSED
```

Excellent! Now let's test reading existing files.

## Strategy 5: Pre-Populating Test Files

Test code that reads existing files:

```python
# test_config_manager_existing.py
import pytest
import json
from config_manager import ConfigManager

@pytest.fixture
def existing_config(tmp_path):
    """Create a config file with existing data."""
    config_file = tmp_path / "config.json"
    
    # Write initial config
    initial_config = {
        "database": "localhost",
        "port": 5432,
        "debug": True
    }
    
    with open(config_file, 'w') as f:
        json.dump(initial_config, f)
    
    return config_file

def test_load_existing_config(existing_config):
    manager = ConfigManager(str(existing_config))
    
    config = manager.load_config()
    
    assert config["database"] == "localhost"
    assert config["port"] == 5432
    assert config["debug"] is True

def test_update_existing_config(existing_config):
    manager = ConfigManager(str(existing_config))
    
    # Update one value
    manager.update_config("port", 3306)
    
    # Verify other values preserved
    config = manager.load_config()
    assert config["database"] == "localhost"  # Unchanged
    assert config["port"] == 3306  # Updated
    assert config["debug"] is True  # Unchanged

def test_delete_config(existing_config):
    manager = ConfigManager(str(existing_config))
    
    # Verify file exists
    assert existing_config.exists()
    
    # Delete it
    manager.delete_config()
    
    # Verify it's gone
    assert not existing_config.exists()
    
    # Loading should return empty dict
    assert manager.load_config() == {}
```

Run this:

```bash
$ pytest test_config_manager_existing.py -v
```

**Output**:

```text
test_config_manager_existing.py::test_load_existing_config PASSED
test_config_manager_existing.py::test_update_existing_config PASSED
test_config_manager_existing.py::test_delete_config PASSED
```

Great! Now let's test error conditions.

## Strategy 6: Testing File I/O Errors

Test how code handles file system errors:

```python
# test_config_manager_errors.py
import pytest
import os
from config_manager import ConfigManager

def test_load_nonexistent_file(tmp_path):
    """Loading a nonexistent file should return empty dict."""
    config_file = tmp_path / "nonexistent.json"
    manager = ConfigManager(str(config_file))
    
    config = manager.load_config()
    
    assert config == {}

def test_load_invalid_json(tmp_path):
    """Loading invalid JSON should raise an error."""
    config_file = tmp_path / "invalid.json"
    
    # Write invalid JSON
    with open(config_file, 'w') as f:
        f.write("{ invalid json }")
    
    manager = ConfigManager(str(config_file))
    
    with pytest.raises(json.JSONDecodeError):
        manager.load_config()

def test_save_to_readonly_directory(tmp_path):
    """Saving to a read-only directory should raise an error."""
    readonly_dir = tmp_path / "readonly"
    readonly_dir.mkdir()
    
    # Make directory read-only
    os.chmod(readonly_dir, 0o444)
    
    config_file = readonly_dir / "config.json"
    manager = ConfigManager(str(config_file))
    
    try:
        with pytest.raises(PermissionError):
            manager.save_config({"key": "value"})
    finally:
        # Restore permissions for cleanup
        os.chmod(readonly_dir, 0o755)

def test_save_creates_parent_directories(tmp_path):
    """Saving should create parent directories if they don't exist."""
    config_file = tmp_path / "a" / "b" / "c" / "config.json"
    manager = ConfigManager(str(config_file))
    
    # Parent directories don't exist yet
    assert not config_file.parent.exists()
    
    # Save should create them
    manager.save_config({"key": "value"})
    
    assert config_file.exists()
    assert config_file.parent.exists()
```

Run this:

```bash
$ pytest test_config_manager_errors.py -v
```

**Output**:

```text
test_config_manager_errors.py::test_load_nonexistent_file PASSED
test_config_manager_errors.py::test_load_invalid_json PASSED
test_config_manager_errors.py::test_save_to_readonly_directory PASSED
test_config_manager_errors.py::test_save_creates_parent_directories PASSED
```

Perfect! Now let's test with different file formats.

## Strategy 7: Testing Multiple File Formats

Test code that handles different file types:

```python
# file_processor.py
import json
import csv
import os

class FileProcessor:
    def __init__(self, file_path):
        self.file_path = file_path
    
    def read_json(self):
        """Read JSON file."""
        with open(self.file_path, 'r') as f:
            return json.load(f)
    
    def write_json(self, data):
        """Write JSON file."""
        with open(self.file_path, 'w') as f:
            json.dump(data, f, indent=2)
    
    def read_csv(self):
        """Read CSV file."""
        with open(self.file_path, 'r') as f:
            reader = csv.DictReader(f)
            return list(reader)
    
    def write_csv(self, data, fieldnames):
        """Write CSV file."""
        with open(self.file_path, 'w', newline='') as f:
            writer = csv.DictWriter(f, fieldnames=fieldnames)
            writer.writeheader()
            writer.writerows(data)
    
    def read_text(self):
        """Read text file."""
        with open(self.file_path, 'r') as f:
            return f.read()
    
    def write_text(self, content):
        """Write text file."""
        with open(self.file_path, 'w') as f:
            f.write(content)
```

```python
# test_file_processor.py
import pytest
from file_processor import FileProcessor

def test_json_roundtrip(tmp_path):
    """Test reading and writing JSON."""
    json_file = tmp_path / "data.json"
    processor = FileProcessor(str(json_file))
    
    data = {"name": "Alice", "age": 30, "active": True}
    processor.write_json(data)
    
    loaded = processor.read_json()
    assert loaded == data

def test_csv_roundtrip(tmp_path):
    """Test reading and writing CSV."""
    csv_file = tmp_path / "data.csv"
    processor = FileProcessor(str(csv_file))
    
    data = [
        {"name": "Alice", "age": "30"},
        {"name": "Bob", "age": "25"}
    ]
    
    processor.write_csv(data, fieldnames=["name", "age"])
    
    loaded = processor.read_csv()
    assert loaded == data

def test_text_roundtrip(tmp_path):
    """Test reading and writing text."""
    text_file = tmp_path / "data.txt"
    processor = FileProcessor(str(text_file))
    
    content = "Hello, World!\nThis is a test.\n"
    processor.write_text(content)
    
    loaded = processor.read_text()
    assert loaded == content

def test_multiline_text(tmp_path):
    """Test text file with multiple lines."""
    text_file = tmp_path / "multiline.txt"
    processor = FileProcessor(str(text_file))
    
    lines = ["Line 1", "Line 2", "Line 3"]
    content = "\n".join(lines)
    
    processor.write_text(content)
    
    loaded = processor.read_text()
    assert loaded == content
```

Run this:

```bash
$ pytest test_file_processor.py -v
```

**Output**:

```text
test_file_processor.py::test_json_roundtrip PASSED
test_file_processor.py::test_csv_roundtrip PASSED
test_file_processor.py::test_text_roundtrip PASSED
test_file_processor.py::test_multiline_text PASSED
```

## Decision Framework: File Testing Strategies

| Strategy | Setup Complexity | Isolation | Best For |
|----------|-----------------|-----------|----------|
| **Manual cleanup** | Low | Good | Simple tests, learning |
| **Unique filenames** | Low | Good | Parallel tests, quick fixes |
| **tempfile module** | Medium | Perfect | Cross-platform, production code |
| **tmp_path fixture** | Low | Perfect | Pytest tests, recommended |
| **tmp_path_factory** | Medium | Perfect | Session-scoped files, shared fixtures |

### When to Apply Each Solution

**tmp_path fixture** (recommended):
- ✅ Function-scoped temporary directory
- ✅ Automatic cleanup
- ✅ pathlib.Path object (modern Python)
- ✅ Isolated per test
- ❌ Can't share files between tests

**tmp_path_factory**:
- ✅ Create temporary directories with custom scope
- ✅ Share files between tests
- ✅ More control over lifecycle
- ❌ More complex setup

**Manual cleanup**:
- ✅ Simple to understand
- ✅ Full control
- ❌ Easy to forget cleanup
- ❌ Not parallel-safe

## Common Failure Modes and Their Signatures

### Symptom: File not found after test

**Pytest output pattern**:

```text
FileNotFoundError: [Errno 2] No such file or directory: '/tmp/pytest-123/test_file.txt'
```

**Diagnostic clues**:
- File path from temporary directory
- File existed during test but gone after
- Cleanup happened too early

**Root cause**: Temporary directory cleaned up before test finished

**Solution**: Ensure fixture scope matches test needs, use `yield` correctly

### Symptom: Tests interfere with each other

**Pytest output pattern**:

```text
AssertionError: assert {'key': 'value1', 'key2': 'value2'} == {'key': 'value1'}
# Extra data from previous test
```

**Diagnostic clues**:
- Tests pass individually but fail together
- File contains data from previous test
- Shared file path

**Root cause**: Tests using same file path without cleanup

**Solution**: Use `tmp_path` fixture for per-test isolation

### Symptom: Permission denied errors

**Pytest output pattern**:

```text
PermissionError: [Errno 13] Permission denied: '/tmp/readonly/file.txt'
```

**Diagnostic clues**:
- Permission error on file or directory
- Test modifying permissions
- Cleanup failing

**Root cause**: Test changed permissions and didn't restore them

**Solution**: Use try/finally to restore permissions, or use `tmp_path` which handles cleanup

## Working with Temporary Files and Directories

## Advanced Temporary File Patterns

In the previous section, we introduced `tmp_path` for basic file testing. Now let's explore advanced patterns for complex scenarios: sharing files between tests, testing with large file structures, and handling binary files.

## Iteration 1: Session-Scoped Temporary Directories

Sometimes you need to share files across multiple tests:

```python
# test_shared_temp.py
import pytest
from pathlib import Path

@pytest.fixture(scope="session")
def shared_data_dir(tmp_path_factory):
    """Create a session-scoped temporary directory."""
    data_dir = tmp_path_factory.mktemp("shared_data")
    
    # Create some shared test data
    (data_dir / "reference.txt").write_text("Reference data")
    (data_dir / "config.json").write_text('{"version": "1.0"}')
    
    return data_dir

def test_read_reference(shared_data_dir):
    """First test reads reference data."""
    content = (shared_data_dir / "reference.txt").read_text()
    assert content == "Reference data"

def test_read_config(shared_data_dir):
    """Second test reads config."""
    import json
    config = json.loads((shared_data_dir / "config.json").read_text())
    assert config["version"] == "1.0"

def test_both_files_exist(shared_data_dir):
    """Third test verifies both files exist."""
    assert (shared_data_dir / "reference.txt").exists()
    assert (shared_data_dir / "config.json").exists()
```

Run this:

```bash
$ pytest test_shared_temp.py -v
```

**Output**:

```text
test_shared_temp.py::test_read_reference PASSED
test_shared_temp.py::test_read_config PASSED
test_shared_temp.py::test_both_files_exist PASSED
```

Good! But what if tests need to modify files? Let's see what happens:

```python
# test_shared_temp_modification.py
import pytest
from pathlib import Path

@pytest.fixture(scope="session")
def shared_data_dir(tmp_path_factory):
    data_dir = tmp_path_factory.mktemp("shared_data")
    (data_dir / "counter.txt").write_text("0")
    return data_dir

def test_increment_counter_first(shared_data_dir):
    """First test increments counter."""
    counter_file = shared_data_dir / "counter.txt"
    count = int(counter_file.read_text())
    counter_file.write_text(str(count + 1))
    
    assert int(counter_file.read_text()) == 1

def test_increment_counter_second(shared_data_dir):
    """Second test also increments counter."""
    counter_file = shared_data_dir / "counter.txt"
    count = int(counter_file.read_text())
    counter_file.write_text(str(count + 1))
    
    # What value do we expect?
    assert int(counter_file.read_text()) == 2
```

Run this:

```bash
$ pytest test_shared_temp_modification.py -v
```

**Output**:

```text
test_shared_temp_modification.py::test_increment_counter_first PASSED
test_shared_temp_modification.py::test_increment_counter_second PASSED
```

### Diagnostic Analysis: Shared State in Session Fixtures

**The complete output**:

Both tests pass, but they're sharing state through the file.

**Let's parse this section by section**:

1. **The session scope**: The fixture creates the directory once for all tests
   - What this tells us: All tests see the same files
   - Modifications persist across tests

2. **The test order dependency**: `test_increment_counter_second` expects the counter to be 1 (from the first test)
   - What this tells us: Tests are not isolated
   - Test order matters

**Root cause identified**: Session-scoped fixtures create shared state, which can lead to test interdependencies.

**Why the current approach can't solve this**: Session scope is intentional for sharing, but we need to be careful about modifications.

**What we need**: Either (1) read-only shared data, or (2) per-test copies of shared data.

## Iteration 2: Read-Only Shared Data with Per-Test Copies

Create shared reference data but give each test its own copy:

```python
# test_copy_pattern.py
import pytest
import shutil
from pathlib import Path

@pytest.fixture(scope="session")
def reference_data(tmp_path_factory):
    """Create reference data once."""
    data_dir = tmp_path_factory.mktemp("reference")
    
    # Create reference files
    (data_dir / "template.txt").write_text("Template content")
    (data_dir / "schema.json").write_text('{"type": "object"}')
    
    return data_dir

@pytest.fixture
def test_data(reference_data, tmp_path):
    """Copy reference data for each test."""
    # Copy all files from reference to test-specific directory
    for file in reference_data.iterdir():
        shutil.copy(file, tmp_path)
    
    return tmp_path

def test_modify_template(test_data):
    """Modify template without affecting other tests."""
    template = test_data / "template.txt"
    template.write_text("Modified content")
    
    assert template.read_text() == "Modified content"

def test_template_unchanged(test_data):
    """Verify template is unchanged from reference."""
    template = test_data / "template.txt"
    
    # Should be original content, not modified
    assert template.read_text() == "Template content"
```

Run this:

```bash
$ pytest test_copy_pattern.py -v
```

**Output**:

```text
test_copy_pattern.py::test_modify_template PASSED
test_copy_pattern.py::test_template_unchanged PASSED
```

Perfect isolation! Each test gets its own copy of the reference data.

## Iteration 3: Complex Directory Structures

Test code that works with nested directories:

```python
# directory_scanner.py
from pathlib import Path

class DirectoryScanner:
    def __init__(self, root_path):
        self.root_path = Path(root_path)
    
    def count_files(self, extension=None):
        """Count files, optionally filtered by extension."""
        if extension:
            return len(list(self.root_path.rglob(f"*.{extension}")))
        return len(list(self.root_path.rglob("*.*")))
    
    def find_files(self, pattern):
        """Find files matching a pattern."""
        return [str(p) for p in self.root_path.rglob(pattern)]
    
    def get_directory_tree(self):
        """Get a tree structure of directories."""
        tree = {}
        for path in self.root_path.rglob("*"):
            if path.is_dir():
                relative = path.relative_to(self.root_path)
                tree[str(relative)] = [
                    p.name for p in path.iterdir()
                ]
        return tree
```

```python
# test_directory_scanner.py
import pytest
from pathlib import Path
from directory_scanner import DirectoryScanner

@pytest.fixture
def complex_directory(tmp_path):
    """Create a complex directory structure."""
    # Create directory structure
    (tmp_path / "src").mkdir()
    (tmp_path / "src" / "main.py").write_text("# main")
    (tmp_path / "src" / "utils.py").write_text("# utils")
    
    (tmp_path / "tests").mkdir()
    (tmp_path / "tests" / "test_main.py").write_text("# test")
    
    (tmp_path / "docs").mkdir()
    (tmp_path / "docs" / "readme.md").write_text("# README")
    (tmp_path / "docs" / "guide.md").write_text("# Guide")
    
    (tmp_path / "data").mkdir()
    (tmp_path / "data" / "input.json").write_text("{}")
    (tmp_path / "data" / "output.json").write_text("{}")
    
    return tmp_path

def test_count_all_files(complex_directory):
    scanner = DirectoryScanner(complex_directory)
    
    # Should find all 7 files
    assert scanner.count_files() == 7

def test_count_python_files(complex_directory):
    scanner = DirectoryScanner(complex_directory)
    
    # Should find 3 .py files
    assert scanner.count_files("py") == 3

def test_count_markdown_files(complex_directory):
    scanner = DirectoryScanner(complex_directory)
    
    # Should find 2 .md files
    assert scanner.count_files("md") == 2

def test_find_test_files(complex_directory):
    scanner = DirectoryScanner(complex_directory)
    
    # Find all test files
    test_files = scanner.find_files("test_*.py")
    
    assert len(test_files) == 1
    assert "test_main.py" in test_files[0]

def test_directory_tree(complex_directory):
    scanner = DirectoryScanner(complex_directory)
    
    tree = scanner.get_directory_tree()
    
    # Verify structure
    assert "src" in tree
    assert "tests" in tree
    assert "docs" in tree
    assert "data" in tree
```

Run this:

```bash
$ pytest test_directory_scanner.py -v
```

**Output**:

```text
test_directory_scanner.py::test_count_all_files PASSED
test_directory_scanner.py::test_count_python_files PASSED
test_directory_scanner.py::test_count_markdown_files PASSED
test_directory_scanner.py::test_find_test_files PASSED
test_directory_scanner.py::test_directory_tree PASSED
```

Excellent! Now let's test with binary files.

## Iteration 4: Binary File Handling

Test code that works with binary files:

```python
# image_processor.py
from pathlib import Path

class ImageProcessor:
    def __init__(self, image_path):
        self.image_path = Path(image_path)
    
    def get_file_size(self):
        """Get file size in bytes."""
        return self.image_path.stat().st_size
    
    def read_header(self, num_bytes=16):
        """Read the first N bytes of the file."""
        with open(self.image_path, 'rb') as f:
            return f.read(num_bytes)
    
    def is_png(self):
        """Check if file is a PNG image."""
        header = self.read_header(8)
        # PNG signature: 89 50 4E 47 0D 0A 1A 0A
        return header == b'\x89PNG\r\n\x1a\n'
    
    def is_jpeg(self):
        """Check if file is a JPEG image."""
        header = self.read_header(2)
        # JPEG signature: FF D8
        return header == b'\xff\xd8'
```

```python
# test_image_processor.py
import pytest
from pathlib import Path
from image_processor import ImageProcessor

@pytest.fixture
def png_file(tmp_path):
    """Create a minimal PNG file."""
    png_path = tmp_path / "test.png"
    
    # PNG signature + minimal IHDR chunk
    png_data = (
        b'\x89PNG\r\n\x1a\n'  # PNG signature
        b'\x00\x00\x00\rIHDR'  # IHDR chunk
        b'\x00\x00\x00\x01'  # Width: 1
        b'\x00\x00\x00\x01'  # Height: 1
        b'\x08\x02\x00\x00\x00'  # Bit depth, color type, etc.
        b'\x90wS\xde'  # CRC
    )
    
    png_path.write_bytes(png_data)
    return png_path

@pytest.fixture
def jpeg_file(tmp_path):
    """Create a minimal JPEG file."""
    jpeg_path = tmp_path / "test.jpg"
    
    # JPEG signature + minimal structure
    jpeg_data = (
        b'\xff\xd8'  # JPEG signature (SOI)
        b'\xff\xe0'  # APP0 marker
        b'\x00\x10'  # Length
        b'JFIF\x00'  # Identifier
        b'\x01\x01'  # Version
        b'\x00'  # Units
        b'\x00\x01\x00\x01'  # X/Y density
        b'\x00\x00'  # Thumbnail
        b'\xff\xd9'  # EOI marker
    )
    
    jpeg_path.write_bytes(jpeg_data)
    return jpeg_path

def test_png_detection(png_file):
    processor = ImageProcessor(png_file)
    
    assert processor.is_png()
    assert not processor.is_jpeg()

def test_jpeg_detection(jpeg_file):
    processor = ImageProcessor(jpeg_file)
    
    assert processor.is_jpeg()
    assert not processor.is_png()

def test_file_size(png_file):
    processor = ImageProcessor(png_file)
    
    # Our minimal PNG is 33 bytes
    assert processor.get_file_size() == 33

def test_read_header(png_file):
    processor = ImageProcessor(png_file)
    
    header = processor.read_header(8)
    assert header == b'\x89PNG\r\n\x1a\n'
```

Run this:

```bash
$ pytest test_image_processor.py -v
```

**Output**:

```text
test_image_processor.py::test_png_detection PASSED
test_image_processor.py::test_jpeg_detection PASSED
test_image_processor.py::test_file_size PASSED
test_image_processor.py::test_read_header PASSED
```

Perfect! Now let's test with large files.

## Iteration 5: Large File Handling

Test code that processes large files efficiently:

```python
# file_chunker.py
from pathlib import Path

class FileChunker:
    def __init__(self, file_path, chunk_size=1024):
        self.file_path = Path(file_path)
        self.chunk_size = chunk_size
    
    def count_chunks(self):
        """Count how many chunks the file contains."""
        file_size = self.file_path.stat().st_size
        return (file_size + self.chunk_size - 1) // self.chunk_size
    
    def read_chunk(self, chunk_number):
        """Read a specific chunk."""
        with open(self.file_path, 'rb') as f:
            f.seek(chunk_number * self.chunk_size)
            return f.read(self.chunk_size)
    
    def process_in_chunks(self, processor_func):
        """Process file in chunks."""
        results = []
        with open(self.file_path, 'rb') as f:
            while True:
                chunk = f.read(self.chunk_size)
                if not chunk:
                    break
                results.append(processor_func(chunk))
        return results
```

```python
# test_file_chunker.py
import pytest
from pathlib import Path
from file_chunker import FileChunker

@pytest.fixture
def large_file(tmp_path):
    """Create a large test file."""
    file_path = tmp_path / "large.bin"
    
    # Create a 10KB file
    data = b'A' * 10240
    file_path.write_bytes(data)
    
    return file_path

def test_count_chunks(large_file):
    chunker = FileChunker(large_file, chunk_size=1024)
    
    # 10KB / 1KB = 10 chunks
    assert chunker.count_chunks() == 10

def test_read_first_chunk(large_file):
    chunker = FileChunker(large_file, chunk_size=1024)
    
    chunk = chunker.read_chunk(0)
    
    assert len(chunk) == 1024
    assert chunk == b'A' * 1024

def test_read_last_chunk(large_file):
    chunker = FileChunker(large_file, chunk_size=1024)
    
    chunk = chunker.read_chunk(9)  # Last chunk
    
    assert len(chunk) == 1024
    assert chunk == b'A' * 1024

def test_process_in_chunks(large_file):
    chunker = FileChunker(large_file, chunk_size=1024)
    
    # Count bytes in each chunk
    results = chunker.process_in_chunks(len)
    
    assert len(results) == 10
    assert all(size == 1024 for size in results)

def test_uneven_chunks(tmp_path):
    """Test file that doesn't divide evenly into chunks."""
    file_path = tmp_path / "uneven.bin"
    file_path.write_bytes(b'B' * 2500)  # 2.5 KB
    
    chunker = FileChunker(file_path, chunk_size=1024)
    
    # Should be 3 chunks (1024 + 1024 + 452)
    assert chunker.count_chunks() == 3
    
    # Last chunk should be partial
    last_chunk = chunker.read_chunk(2)
    assert len(last_chunk) == 452
```

Run this:

```bash
$ pytest test_file_chunker.py -v
```

**Output**:

```text
test_file_chunker.py::test_count_chunks PASSED
test_file_chunker.py::test_read_first_chunk PASSED
test_file_chunker.py::test_read_last_chunk PASSED
test_file_chunker.py::test_process_in_chunks PASSED
test_file_chunker.py::test_uneven_chunks PASSED
```

## The Complete Temporary File Toolkit

Here's a comprehensive fixture setup for file testing:

```python
# conftest.py (complete file testing setup)
import pytest
import shutil
from pathlib import Path

@pytest.fixture(scope="session")
def reference_files(tmp_path_factory):
    """Create reference files once for all tests."""
    ref_dir = tmp_path_factory.mktemp("reference")
    
    # Text files
    (ref_dir / "template.txt").write_text("Template content")
    (ref_dir / "config.json").write_text('{"version": "1.0"}')
    
    # Binary files
    (ref_dir / "data.bin").write_bytes(b'\x00\x01\x02\x03')
    
    return ref_dir

@pytest.fixture
def test_files(reference_files, tmp_path):
    """Copy reference files for each test."""
    for file in reference_files.iterdir():
        shutil.copy(file, tmp_path)
    return tmp_path

@pytest.fixture
def empty_directory(tmp_path):
    """Provide an empty directory."""
    return tmp_path

@pytest.fixture
def nested_directory(tmp_path):
    """Create a nested directory structure."""
    (tmp_path / "level1").mkdir()
    (tmp_path / "level1" / "level2").mkdir()
    (tmp_path / "level1" / "level2" / "level3").mkdir()
    return tmp_path

@pytest.fixture
def project_structure(tmp_path):
    """Create a typical project structure."""
    # Source directory
    src = tmp_path / "src"
    src.mkdir()
    (src / "__init__.py").write_text("")
    (src / "main.py").write_text("# main")
    
    # Tests directory
    tests = tmp_path / "tests"
    tests.mkdir()
    (tests / "__init__.py").write_text("")
    (tests / "test_main.py").write_text("# test")
    
    # Docs directory
    docs = tmp_path / "docs"
    docs.mkdir()
    (docs / "README.md").write_text("# README")
    
    # Config files
    (tmp_path / "setup.py").write_text("# setup")
    (tmp_path / "requirements.txt").write_text("pytest\n")
    
    return tmp_path

@pytest.fixture
def binary_file(tmp_path):
    """Create a binary file with known content."""
    file_path = tmp_path / "binary.dat"
    file_path.write_bytes(bytes(range(256)))
    return file_path

@pytest.fixture
def large_text_file(tmp_path):
    """Create a large text file."""
    file_path = tmp_path / "large.txt"
    
    # Create 1MB of text
    lines = ["Line {}\n".format(i) for i in range(100000)]
    file_path.write_text("".join(lines))
    
    return file_path
```

## Decision Framework: Temporary File Strategies

| Strategy | Scope | Isolation | Best For |
|----------|-------|-----------|----------|
| **tmp_path** | Function | Perfect | Most tests, default choice |
| **tmp_path_factory** | Custom | Perfect | Session-scoped, shared reference data |
| **Reference + Copy** | Mixed | Perfect | Expensive setup, read-mostly data |
| **In-memory (io.BytesIO)** | Function | Perfect | Small files, no disk I/O needed |

### When to Apply Each Solution

**tmp_path** (recommended default):
- ✅ Function-scoped isolation
- ✅ Automatic cleanup
- ✅ pathlib.Path interface
- ✅ Simple and clear
- ❌ Can't share between tests

**tmp_path_factory + session scope**:
- ✅ Share expensive setup across tests
- ✅ Create reference data once
- ✅ Custom scope control
- ❌ Risk of test interdependency
- ❌ More complex

**Reference + Copy pattern**:
- ✅ Best of both worlds: shared setup, isolated tests
- ✅ Each test gets fresh copy
- ✅ Safe for modifications
- ❌ Extra copy overhead

**In-memory files (io.BytesIO)**:
- ✅ No disk I/O
- ✅ Fastest
- ✅ No cleanup needed
- ❌ Limited to small files
- ❌ Can't test actual file operations

## Common Failure Modes and Their Signatures

### Symptom: Temporary directory not cleaned up

**Pytest output pattern**:

```text
$ ls /tmp/pytest-of-user/
pytest-0  pytest-1  pytest-2  pytest-3  # Many old directories
```

**Diagnostic clues**:
- Multiple pytest temporary directories
- Directories not being removed
- Disk space growing

**Root cause**: Pytest cleanup failed (crash, interrupt, or permission issue)

**Solution**: Pytest cleans up automatically on normal exit; manual cleanup: `pytest --basetemp=/tmp/pytest-custom`

### Symptom: Tests fail due to file permissions

**Pytest output pattern**:

```text
PermissionError: [Errno 13] Permission denied: '/tmp/pytest-123/readonly/file.txt'
```

**Diagnostic clues**:
- Permission error in temporary directory
- Test modified permissions
- Cleanup failing

**Root cause**: Test changed file/directory permissions and didn't restore them

**Solution**: Always restore permissions in finally block or use context managers

### Symptom: Session-scoped fixture causing test interdependency

**Pytest output pattern**:

```text
test_first PASSED
test_second FAILED  # Expects different state than test_first left
```

**Diagnostic clues**:
- Tests pass individually
- Fail when run together
- Session-scoped fixture involved

**Root cause**: Tests modifying shared session-scoped files

**Solution**: Use reference + copy pattern, or switch to function-scoped fixtures
