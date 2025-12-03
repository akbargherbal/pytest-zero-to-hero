# Chapter 20: Pro Tips, Best Practices, and Common Pitfalls

## Time-Saving Tips

You've mastered pytest's core features. Now let's explore productivity multipliers that professional developers use daily. These aren't just conveniences—they're force multipliers that transform how you work with tests.

We'll build our examples around a realistic scenario: a web API client library with multiple endpoints, authentication, error handling, and data validation. This gives us enough complexity to demonstrate where these tools shine.

```python
# src/api_client.py
import requests
from typing import Dict, List, Optional
import time

class APIClient:
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url
        self.api_key = api_key
        self.session = requests.Session()
        self.session.headers.update({"Authorization": f"Bearer {api_key}"})
    
    def get_user(self, user_id: int) -> Dict:
        """Fetch user by ID."""
        response = self.session.get(f"{self.base_url}/users/{user_id}")
        response.raise_for_status()
        return response.json()
    
    def list_users(self, page: int = 1, limit: int = 10) -> List[Dict]:
        """List users with pagination."""
        response = self.session.get(
            f"{self.base_url}/users",
            params={"page": page, "limit": limit}
        )
        response.raise_for_status()
        return response.json()
    
    def create_user(self, username: str, email: str) -> Dict:
        """Create a new user."""
        response = self.session.post(
            f"{self.base_url}/users",
            json={"username": username, "email": email}
        )
        response.raise_for_status()
        return response.json()
    
    def update_user(self, user_id: int, **kwargs) -> Dict:
        """Update user fields."""
        response = self.session.patch(
            f"{self.base_url}/users/{user_id}",
            json=kwargs
        )
        response.raise_for_status()
        return response.json()
    
    def delete_user(self, user_id: int) -> None:
        """Delete a user."""
        response = self.session.delete(f"{self.base_url}/users/{user_id}")
        response.raise_for_status()
    
    def search_users(self, query: str) -> List[Dict]:
        """Search users by query string."""
        time.sleep(0.5)  # Simulate slow search
        response = self.session.get(
            f"{self.base_url}/users/search",
            params={"q": query}
        )
        response.raise_for_status()
        return response.json()
```

Our initial test suite covers the basic functionality:

```python
# tests/test_api_client.py
import pytest
from unittest.mock import Mock, patch
from src.api_client import APIClient

@pytest.fixture
def mock_session():
    with patch('src.api_client.requests.Session') as mock:
        session = Mock()
        mock.return_value = session
        yield session

@pytest.fixture
def client(mock_session):
    return APIClient("https://api.example.com", "test-key")

def test_get_user_returns_user_data(client, mock_session):
    mock_session.get.return_value.json.return_value = {
        "id": 1,
        "username": "alice",
        "email": "alice@example.com"
    }
    
    result = client.get_user(1)
    
    assert result["username"] == "alice"
    mock_session.get.assert_called_once_with("https://api.example.com/users/1")

def test_list_users_with_pagination(client, mock_session):
    mock_session.get.return_value.json.return_value = [
        {"id": 1, "username": "alice"},
        {"id": 2, "username": "bob"}
    ]
    
    result = client.list_users(page=2, limit=5)
    
    assert len(result) == 2
    mock_session.get.assert_called_once_with(
        "https://api.example.com/users",
        params={"page": 2, "limit": 5}
    )

def test_create_user_sends_correct_data(client, mock_session):
    mock_session.post.return_value.json.return_value = {
        "id": 3,
        "username": "charlie",
        "email": "charlie@example.com"
    }
    
    result = client.create_user("charlie", "charlie@example.com")
    
    assert result["id"] == 3
    mock_session.post.assert_called_once_with(
        "https://api.example.com/users",
        json={"username": "charlie", "email": "charlie@example.com"}
    )

def test_update_user_patches_fields(client, mock_session):
    mock_session.patch.return_value.json.return_value = {
        "id": 1,
        "username": "alice_updated"
    }
    
    result = client.update_user(1, username="alice_updated")
    
    assert result["username"] == "alice_updated"

def test_delete_user_calls_correct_endpoint(client, mock_session):
    client.delete_user(1)
    
    mock_session.delete.assert_called_once_with("https://api.example.com/users/1")

def test_search_users_returns_matching_results(client, mock_session):
    mock_session.get.return_value.json.return_value = [
        {"id": 1, "username": "alice"}
    ]
    
    result = client.search_users("ali")
    
    assert len(result) == 1
    assert result[0]["username"] == "alice"
```

This test suite works, but running it repeatedly during development is tedious. Let's see how professional developers optimize their workflow.

## The Manual Testing Loop Problem

During active development, you make a change, switch to terminal, run pytest, read results, switch back to editor, make another change... This context switching kills flow state.

Let's see what happens when we're actively developing a new feature—adding rate limiting to our API client:

```python
# src/api_client.py (adding rate limiting)
import requests
from typing import Dict, List, Optional
import time
from datetime import datetime, timedelta

class APIClient:
    def __init__(self, base_url: str, api_key: str, rate_limit: int = 10):
        self.base_url = base_url
        self.api_key = api_key
        self.rate_limit = rate_limit
        self.request_times = []
        self.session = requests.Session()
        self.session.headers.update({"Authorization": f"Bearer {api_key}"})
    
    def _check_rate_limit(self):
        """Enforce rate limiting."""
        now = datetime.now()
        # Remove requests older than 1 minute
        self.request_times = [
            t for t in self.request_times 
            if now - t < timedelta(minutes=1)
        ]
        
        if len(self.request_times) >= self.rate_limit:
            oldest = self.request_times[0]
            wait_time = 60 - (now - oldest).total_seconds()
            raise Exception(f"Rate limit exceeded. Wait {wait_time:.1f}s")
        
        self.request_times.append(now)
    
    def get_user(self, user_id: int) -> Dict:
        self._check_rate_limit()
        response = self.session.get(f"{self.base_url}/users/{user_id}")
        response.raise_for_status()
        return response.json()
```

We write a test for the new rate limiting feature:

```python
# tests/test_api_client.py (adding rate limit test)
def test_rate_limit_enforced(client, mock_session):
    """Test that rate limiting prevents excessive requests."""
    mock_session.get.return_value.json.return_value = {"id": 1}
    
    # Make requests up to the limit
    for i in range(10):
        client.get_user(i)
    
    # The 11th request should fail
    with pytest.raises(Exception, match="Rate limit exceeded"):
        client.get_user(11)
```

Run the test:

```bash
$ pytest tests/test_api_client.py::test_rate_limit_enforced -v
```

**Output**:

```text
tests/test_api_client.py::test_rate_limit_enforced PASSED
```

Good! But now we realize the error message should be more specific. We update the code:

```python
# src/api_client.py (improved error message)
def _check_rate_limit(self):
    now = datetime.now()
    self.request_times = [
        t for t in self.request_times 
        if now - t < timedelta(minutes=1)
    ]
    
    if len(self.request_times) >= self.rate_limit:
        oldest = self.request_times[0]
        wait_time = 60 - (now - oldest).total_seconds()
        raise RateLimitError(
            f"Rate limit of {self.rate_limit} requests/minute exceeded. "
            f"Retry after {wait_time:.1f} seconds"
        )
    
    self.request_times.append(now)
```

Now we need to:
1. Switch to terminal
2. Press up arrow to get previous command
3. Press enter
4. Wait for results
5. Switch back to editor

Repeat this 20 times while refining the feature. The friction adds up.

### Iteration 1: Automatic Test Watching

Install pytest-watch:

```bash
$ pip install pytest-watch
```

Now start the watcher:

```bash
$ ptw tests/test_api_client.py::test_rate_limit_enforced -- -v
```

**What happens**:

pytest-watch monitors your files. Every time you save a change to either the test file or the source code, it automatically reruns the tests. No manual intervention needed.

**Initial output**:

```text
[PYTEST-WATCH] Running: pytest tests/test_api_client.py::test_rate_limit_enforced -v

tests/test_api_client.py::test_rate_limit_enforced PASSED

[PYTEST-WATCH] Watching for changes...
```

You make a change to the error message. Save the file. Instantly:

```text
[PYTEST-WATCH] Change detected: src/api_client.py
[PYTEST-WATCH] Running: pytest tests/test_api_client.py::test_rate_limit_enforced -v

tests/test_api_client.py::test_rate_limit_enforced FAILED

E       AssertionError: Pattern 'Rate limit exceeded' not found in 'Rate limit of 10 requests/minute exceeded. Retry after 59.8 seconds'

[PYTEST-WATCH] Watching for changes...
```

The test failed because we changed the error message but didn't update the test. Update the test:

```python
def test_rate_limit_enforced(client, mock_session):
    mock_session.get.return_value.json.return_value = {"id": 1}
    
    for i in range(10):
        client.get_user(i)
    
    with pytest.raises(RateLimitError, match="Rate limit of 10 requests/minute exceeded"):
        client.get_user(11)
```

Save. Instantly:

```text
[PYTEST-WATCH] Change detected: tests/test_api_client.py
[PYTEST-WATCH] Running: pytest tests/test_api_client.py::test_rate_limit_enforced -v

tests/test_api_client.py::test_rate_limit_enforced PASSED

[PYTEST-WATCH] Watching for changes...
```

**The workflow transformation**:

- **Before**: Edit → Switch → Command → Wait → Read → Switch → Edit (6 steps)
- **After**: Edit → Read (2 steps)

You stay in your editor. The feedback appears in a terminal window you glance at. Your flow state remains intact.

### Advanced pytest-watch Patterns

**Watch specific test patterns**:

```bash
# Watch all tests matching a pattern
$ ptw tests/ -k rate_limit -- -v

# Watch and run only failed tests on next run
$ ptw tests/ -- -v --lf

# Watch with coverage
$ ptw tests/ -- --cov=src --cov-report=term-missing
```

**Clear screen between runs** (reduces visual clutter):

```bash
$ ptw tests/ --clear -- -v
```

**Run on-pass and on-fail commands** (e.g., notifications):

```bash
# macOS notification on failure
$ ptw tests/ --onpass "echo '✓ Tests passed'" --onfail "osascript -e 'display notification \"Tests failed\" with title \"pytest\"'"
```

**Ignore specific paths** (e.g., don't rerun on documentation changes):

```bash
$ ptw tests/ --ignore ./docs --ignore ./build
```

### When to Use pytest-watch

**Optimal scenarios**:
- Active feature development with tight feedback loops
- Refactoring existing code with comprehensive tests
- TDD workflows where you write test first, then implementation
- Debugging failing tests where you're making incremental fixes

**When to avoid**:
- Slow test suites (>10 seconds) where constant reruns are disruptive
- Tests with external dependencies that can't be mocked (database migrations, API calls)
- When you need to think deeply between changes without immediate feedback

**Pro tip**: Use pytest-watch for the specific test you're working on, not the entire suite. This keeps feedback instant:

```bash
# Good: Fast, focused feedback
$ ptw tests/test_api_client.py::test_rate_limit_enforced -- -v

# Less good: Slow, unfocused feedback
$ ptw tests/ -- -v
```

## Parallel Test Execution with pytest-xdist

## The Sequential Execution Bottleneck

Our API client test suite has grown. We now have 50+ tests covering all endpoints, error cases, and edge conditions. Running them sequentially takes time:

```bash
$ pytest tests/test_api_client.py -v
```

**Output**:

```text
tests/test_api_client.py::test_get_user_returns_user_data PASSED
tests/test_api_client.py::test_get_user_handles_404 PASSED
tests/test_api_client.py::test_get_user_handles_network_error PASSED
tests/test_api_client.py::test_list_users_with_pagination PASSED
tests/test_api_client.py::test_list_users_empty_result PASSED
tests/test_api_client.py::test_create_user_sends_correct_data PASSED
tests/test_api_client.py::test_create_user_validates_email PASSED
tests/test_api_client.py::test_create_user_handles_duplicate PASSED
... (42 more tests)

======================== 50 passed in 12.34s ========================
```

12 seconds isn't terrible, but it's long enough to break concentration. And this is just one module. A real project might have hundreds of test files.

Let's add some realistic complexity—tests that involve actual timing:

```python
# tests/test_api_client.py (adding slow tests)
def test_search_users_performance(client, mock_session):
    """Verify search completes within acceptable time."""
    mock_session.get.return_value.json.return_value = [
        {"id": i, "username": f"user{i}"} for i in range(100)
    ]
    
    import time
    start = time.time()
    result = client.search_users("user")
    duration = time.time() - start
    
    assert len(result) == 100
    assert duration < 1.0  # Should complete in under 1 second

def test_rate_limit_reset_after_window(client, mock_session):
    """Verify rate limit resets after time window."""
    mock_session.get.return_value.json.return_value = {"id": 1}
    
    # Fill the rate limit
    for i in range(10):
        client.get_user(i)
    
    # Should fail immediately
    with pytest.raises(RateLimitError):
        client.get_user(11)
    
    # Wait for window to pass
    time.sleep(1.1)
    
    # Should succeed now
    client.get_user(12)  # Should not raise

def test_retry_logic_with_backoff(client, mock_session):
    """Test exponential backoff on retries."""
    # Simulate 3 failures then success
    mock_session.get.side_effect = [
        requests.exceptions.ConnectionError(),
        requests.exceptions.ConnectionError(),
        requests.exceptions.ConnectionError(),
        Mock(json=lambda: {"id": 1})
    ]
    
    start = time.time()
    result = client.get_user_with_retry(1, max_retries=3)
    duration = time.time() - start
    
    assert result["id"] == 1
    assert duration > 0.5  # Should have waited during backoff
```

Now our test suite takes significantly longer:

```bash
$ pytest tests/test_api_client.py -v
```

**Output**:

```text
======================== 53 passed in 18.67s ========================
```

18 seconds is too long for rapid iteration. The problem: tests run one at a time, even though most are independent and could run simultaneously.

### Iteration 1: Parallel Execution with pytest-xdist

Install pytest-xdist:

```bash
$ pip install pytest-xdist
```

Run tests in parallel using multiple CPU cores:

```bash
$ pytest tests/test_api_client.py -n auto
```

The `-n auto` flag tells pytest-xdist to automatically detect the number of CPU cores and create that many worker processes.

**Output**:

```text
======================== 53 passed in 6.23s ========================
```

**Result**: 18.67s → 6.23s (3x speedup on a 4-core machine)

### Diagnostic Analysis: How Parallel Execution Works

Let's understand what just happened. Run with verbose output to see the distribution:

```bash
$ pytest tests/test_api_client.py -n 4 -v
```

**Output**:

```text
[gw0] PASSED tests/test_api_client.py::test_get_user_returns_user_data
[gw1] PASSED tests/test_api_client.py::test_list_users_with_pagination
[gw2] PASSED tests/test_api_client.py::test_create_user_sends_correct_data
[gw3] PASSED tests/test_api_client.py::test_update_user_patches_fields
[gw0] PASSED tests/test_api_client.py::test_delete_user_calls_correct_endpoint
[gw1] PASSED tests/test_api_client.py::test_search_users_returns_matching_results
...
```

**What this tells us**:

1. **`[gw0]`, `[gw1]`, etc.**: Gateway workers—separate processes running tests
2. **Distribution**: pytest-xdist distributes tests across workers using a scheduling algorithm
3. **Independence**: Each worker has its own Python interpreter, so tests can't interfere with each other

**The scheduling algorithm**:
- Tests are distributed to workers as they become available
- Slow tests don't block fast tests
- Load balancing happens automatically

### Iteration 2: Handling Test Dependencies

Not all tests can run in parallel. Some have implicit dependencies. Let's see what breaks:

```python
# tests/test_api_client_stateful.py
import pytest
from src.api_client import APIClient

# Shared state (BAD PRACTICE - for demonstration)
_test_user_id = None

def test_create_user_for_subsequent_tests(client, mock_session):
    """Create a user that other tests will use."""
    global _test_user_id
    mock_session.post.return_value.json.return_value = {"id": 999}
    
    result = client.create_user("testuser", "test@example.com")
    _test_user_id = result["id"]
    
    assert _test_user_id == 999

def test_update_created_user(client, mock_session):
    """Update the user created in previous test."""
    global _test_user_id
    assert _test_user_id is not None, "User must be created first"
    
    mock_session.patch.return_value.json.return_value = {
        "id": _test_user_id,
        "username": "updated"
    }
    
    result = client.update_user(_test_user_id, username="updated")
    assert result["username"] == "updated"

def test_delete_created_user(client, mock_session):
    """Delete the user created in first test."""
    global _test_user_id
    assert _test_user_id is not None, "User must be created first"
    
    client.delete_user(_test_user_id)
    mock_session.delete.assert_called_once()
```

Run sequentially—works fine:

```bash
$ pytest tests/test_api_client_stateful.py -v
```

**Output**:

```text
tests/test_api_client_stateful.py::test_create_user_for_subsequent_tests PASSED
tests/test_api_client_stateful.py::test_update_created_user PASSED
tests/test_api_client_stateful.py::test_delete_created_user PASSED

======================== 3 passed in 0.12s ========================
```

Run in parallel—breaks:

```bash
$ pytest tests/test_api_client_stateful.py -n 2 -v
```

**Output**:

```text
[gw0] PASSED tests/test_api_client_stateful.py::test_create_user_for_subsequent_tests
[gw1] FAILED tests/test_api_client_stateful.py::test_update_created_user

E       AssertionError: User must be created first
E       assert None is not None

[gw0] FAILED tests/test_api_client_stateful.py::test_delete_created_user

E       AssertionError: User must be created first
E       assert None is not None

======================== 1 passed, 2 failed in 0.15s ========================
```

**Root cause**: Each worker process has its own memory space. The global variable `_test_user_id` set in `gw0` doesn't exist in `gw1`.

**Solution 1: Make tests independent** (preferred):

```python
# tests/test_api_client_stateful.py (fixed)
def test_update_user_independent(client, mock_session):
    """Update a user (self-contained test)."""
    # Create the user within this test
    user_id = 999
    mock_session.patch.return_value.json.return_value = {
        "id": user_id,
        "username": "updated"
    }
    
    result = client.update_user(user_id, username="updated")
    assert result["username"] == "updated"

def test_delete_user_independent(client, mock_session):
    """Delete a user (self-contained test)."""
    user_id = 999
    client.delete_user(user_id)
    mock_session.delete.assert_called_once_with(
        "https://api.example.com/users/999"
    )
```

**Solution 2: Use fixtures for shared setup**:

```python
# tests/test_api_client_stateful.py (using fixtures)
@pytest.fixture
def created_user_id(client, mock_session):
    """Fixture that creates a user and returns its ID."""
    mock_session.post.return_value.json.return_value = {"id": 999}
    result = client.create_user("testuser", "test@example.com")
    return result["id"]

def test_update_user_with_fixture(client, mock_session, created_user_id):
    """Update a user using fixture."""
    mock_session.patch.return_value.json.return_value = {
        "id": created_user_id,
        "username": "updated"
    }
    
    result = client.update_user(created_user_id, username="updated")
    assert result["username"] == "updated"
```

Now parallel execution works because each test gets its own fixture instance.

### Iteration 3: Controlling Parallelization

Some tests genuinely can't run in parallel—they modify shared resources like databases or files. Mark them:

```python
# tests/test_api_client_integration.py
import pytest

@pytest.mark.serial
def test_database_migration():
    """This test modifies the database schema."""
    # Must run alone
    pass

@pytest.mark.serial
def test_cache_clear():
    """This test clears global cache."""
    # Must run alone
    pass
```

Configure pytest to respect the marker:

```ini
# pytest.ini
[pytest]
markers =
    serial: marks tests that must run serially (not in parallel)
```

Run with a custom hook to enforce serial execution:

```python
# conftest.py
def pytest_collection_modifyitems(config, items):
    """Move serial tests to the end and mark them."""
    serial_tests = []
    parallel_tests = []
    
    for item in items:
        if item.get_closest_marker('serial'):
            serial_tests.append(item)
        else:
            parallel_tests.append(item)
    
    items[:] = parallel_tests + serial_tests
```

Or use pytest-xdist's built-in support:

```bash
# Run parallel tests in parallel, serial tests sequentially
$ pytest -n auto -m "not serial"  # Run parallel tests first
$ pytest -m serial                 # Then run serial tests
```

### Advanced pytest-xdist Patterns

**Load balancing strategies**:

```bash
# Default: distribute tests as workers become available
$ pytest -n auto

# Load group: keep tests from same file together (better for fixtures)
$ pytest -n auto --dist loadgroup

# Load file: distribute entire files to workers
$ pytest -n auto --dist loadfile

# Load scope: distribute by fixture scope
$ pytest -n auto --dist loadscope
```

**Specify exact number of workers**:

```bash
# Use 4 workers regardless of CPU count
$ pytest -n 4

# Use half the available CPUs
$ pytest -n auto --maxprocesses=2
```

**Remote execution** (run tests on multiple machines):

```bash
# Run tests on remote SSH hosts
$ pytest -d --tx ssh=user@host1 --tx ssh=user@host2
```

### Performance Comparison: Real Numbers

Let's measure the actual impact on our full test suite:

```bash
# Sequential execution
$ time pytest tests/ -v
```

**Output**:

```text
======================== 247 passed in 45.23s ========================
real    0m45.234s
```

```bash
# Parallel execution (4 cores)
$ time pytest tests/ -n 4 -v
```

**Output**:

```text
======================== 247 passed in 13.67s ========================
real    0m13.671s
```

**Speedup**: 45.23s → 13.67s (3.3x faster)

**Why not 4x?**: Overhead from process creation, test collection, and result aggregation. The speedup approaches the number of cores as test execution time dominates overhead.

### When to Use pytest-xdist

**Optimal scenarios**:
- Large test suites (>100 tests) where total runtime exceeds 10 seconds
- CI/CD pipelines where test time directly impacts deployment speed
- Tests that are truly independent (no shared state)
- CPU-bound tests (computation, data processing)

**When to avoid**:
- Small test suites (<50 tests) where overhead exceeds benefit
- Tests with shared state that can't be easily isolated
- I/O-bound tests (network, disk) where parallelization doesn't help
- Debugging scenarios where you need predictable execution order

**Cost-benefit analysis**:

| Test Count | Sequential | Parallel (4 cores) | Overhead | Net Benefit |
|------------|------------|-------------------|----------|-------------|
| 10 tests   | 2s         | 1.5s              | 0.5s     | Minimal     |
| 50 tests   | 10s        | 3.5s              | 0.5s     | 6s saved    |
| 200 tests  | 45s        | 13s               | 1s       | 31s saved   |
| 1000 tests | 240s       | 65s               | 5s       | 170s saved  |

**Pro tip**: Use pytest-xdist in CI, but not necessarily during local development. The predictable execution order of sequential tests makes debugging easier.

## Test Selection Shortcuts

## The Full Suite Problem

You're debugging a specific feature—the rate limiting logic. Running the entire test suite to verify your fix is wasteful:

```bash
$ pytest tests/
```

**Output**:

```text
======================== 247 passed in 45.23s ========================
```

You only care about 3 tests related to rate limiting, but you just waited 45 seconds to see their results.

### Iteration 1: Running Specific Tests

**Run a single test file**:

```bash
$ pytest tests/test_api_client.py
```

**Run a specific test function**:

```bash
$ pytest tests/test_api_client.py::test_rate_limit_enforced
```

**Run a specific test class**:

```bash
$ pytest tests/test_api_client.py::TestRateLimiting
```

**Run a specific test method in a class**:

```bash
$ pytest tests/test_api_client.py::TestRateLimiting::test_rate_limit_enforced
```

This works, but requires typing long paths. Let's see better approaches.

### Iteration 2: Keyword-Based Selection

Use `-k` to match test names by pattern:

```bash
# Run all tests with "rate_limit" in the name
$ pytest -k rate_limit
```

**Output**:

```text
collected 247 items / 244 deselected / 3 selected

tests/test_api_client.py::test_rate_limit_enforced PASSED
tests/test_api_client.py::test_rate_limit_reset_after_window PASSED
tests/test_api_client.py::test_rate_limit_custom_window PASSED

======================== 3 passed, 244 deselected in 0.87s ========================
```

**Powerful pattern matching**:

```bash
# Run tests matching multiple keywords (OR logic)
$ pytest -k "rate_limit or retry"

# Run tests matching all keywords (AND logic)
$ pytest -k "rate_limit and reset"

# Exclude tests (NOT logic)
$ pytest -k "rate_limit and not reset"

# Complex expressions
$ pytest -k "(rate_limit or retry) and not slow"
```

**Real-world example**: You're working on authentication. Run all auth-related tests:

```bash
$ pytest -k "auth or login or token"
```

**Output**:

```text
collected 247 items / 231 deselected / 16 selected

tests/test_auth.py::test_login_with_valid_credentials PASSED
tests/test_auth.py::test_login_with_invalid_credentials FAILED
tests/test_auth.py::test_token_generation PASSED
tests/test_auth.py::test_token_expiration PASSED
tests/test_auth.py::test_token_refresh PASSED
...

======================== 15 passed, 1 failed, 231 deselected in 3.21s ========================
```

One test failed. Now you want to focus only on that failure.

### Iteration 3: Running Failed Tests Only

After a test run, pytest remembers which tests failed. Use `--lf` (last failed):

```bash
$ pytest --lf
```

**Output**:

```text
run-last-failure: rerun previous 1 failure

collected 247 items / 246 deselected / 1 selected

tests/test_auth.py::test_login_with_invalid_credentials FAILED

E       AssertionError: Expected 401, got 500

======================== 1 failed, 246 deselected in 0.23s ========================
```

Fix the bug, rerun:

```bash
$ pytest --lf
```

**Output**:

```text
run-last-failure: rerun previous 1 failure

collected 247 items / 246 deselected / 1 selected

tests/test_auth.py::test_login_with_invalid_credentials PASSED

======================== 1 passed, 246 deselected in 0.19s ========================
```

**Run failed tests first, then the rest** (useful for CI):

```bash
$ pytest --ff
```

This runs previously failed tests first, then continues with the rest if they pass.

### Iteration 4: Running New or Modified Tests

Use `--nf` (new first) to prioritize tests in recently modified files:

```bash
$ pytest --nf
```

This is useful when you've just added new tests or modified existing ones.

### Iteration 5: Directory and Path Patterns

**Run all tests in a directory**:

```bash
$ pytest tests/unit/
```

**Run tests matching a path pattern**:

```bash
# Run all test files matching pattern
$ pytest tests/test_api*.py

# Run tests in multiple directories
$ pytest tests/unit/ tests/integration/
```

**Combine path and keyword selection**:

```bash
$ pytest tests/unit/ -k rate_limit
```

### Iteration 6: Marker-Based Selection

We covered markers in Chapter 6, but they're crucial for test selection. Quick recap with practical examples:

```python
# tests/test_api_client.py
import pytest

@pytest.mark.slow
def test_search_users_performance(client, mock_session):
    """This test takes 2+ seconds."""
    pass

@pytest.mark.integration
def test_real_api_call(client):
    """This test hits a real API."""
    pass

@pytest.mark.unit
def test_rate_limit_calculation():
    """Fast unit test."""
    pass
```

**Run only fast tests** (exclude slow ones):

```bash
$ pytest -m "not slow"
```

**Run only integration tests**:

```bash
$ pytest -m integration
```

**Combine markers with boolean logic**:

```bash
# Run unit tests that aren't slow
$ pytest -m "unit and not slow"

# Run integration or slow tests
$ pytest -m "integration or slow"
```

### Iteration 7: The Ultimate Workflow Combination

Here's how professional developers combine these techniques:

**During active development** (fast feedback):

```bash
# Watch specific tests related to current work
$ ptw -k rate_limit -- -v --lf
```

This watches for changes, runs only tests matching "rate_limit", and prioritizes previously failed tests.

**Before committing** (verify your changes):

```bash
# Run tests in modified files, then new tests, then failed tests
$ pytest --nf --ff -v
```

**In CI/CD** (comprehensive validation):

```bash
# Run all tests in parallel, generate coverage report
$ pytest -n auto --cov=src --cov-report=html
```

**Quick smoke test** (verify nothing is broken):

```bash
# Run only fast unit tests
$ pytest -m "unit and not slow" -x
```

The `-x` flag stops on first failure, giving you immediate feedback.

### Advanced Selection Patterns

**Run tests that failed in the last N runs**:

```bash
# Requires pytest-cache plugin (built-in)
$ pytest --lf --last-failed-no-failures all
```

**Run tests in random order** (detect hidden dependencies):

```bash
$ pip install pytest-random-order
$ pytest --random-order
```

**Run tests until one fails** (useful for flaky test detection):

```bash
$ pytest --maxfail=1
```

**Run a test multiple times** (detect intermittent failures):

```bash
$ pip install pytest-repeat
$ pytest tests/test_api_client.py::test_rate_limit_enforced --count=100
```

### Decision Framework: Which Selection Method When?

| Scenario | Command | Why |
|----------|---------|-----|
| Working on specific feature | `pytest -k feature_name` | Fast, intuitive |
| Debugging a failure | `pytest --lf` | Focus on broken code |
| Just added new tests | `pytest --nf` | Verify new code first |
| Pre-commit check | `pytest --ff -x` | Fast fail on regressions |
| CI/CD full validation | `pytest -n auto` | Comprehensive, fast |
| Smoke test | `pytest -m "not slow" -x` | Quick confidence check |
| Specific file/function | `pytest path::test_name` | Precise targeting |

### Pro Tips for Test Selection

**1. Create shell aliases for common patterns**:

```bash
# Add to ~/.bashrc or ~/.zshrc
alias pt='pytest -v'
alias ptf='pytest --lf -v'
alias ptw='ptw -- -v'
alias pts='pytest -m "not slow" -x'
```

**2. Use pytest.ini for default selection**:

```ini
# pytest.ini
[pytest]
# Always exclude slow tests unless explicitly requested
addopts = -m "not slow"
```

Override when needed:

```bash
$ pytest -m ""  # Run all tests, including slow ones
```

**3. Create custom markers for your workflow**:

```python
# conftest.py
def pytest_configure(config):
    config.addinivalue_line("markers", "wip: work in progress tests")
    config.addinivalue_line("markers", "smoke: smoke tests for quick validation")
    config.addinivalue_line("markers", "regression: regression tests for known bugs")
```

Then use them:

```bash
# Run only tests you're actively working on
$ pytest -m wip

# Quick smoke test before commit
$ pytest -m smoke
```

**4. Combine with pytest-watch for ultimate productivity**:

```bash
# Watch and run only failed tests related to auth
$ ptw -k auth -- --lf -v
```

This creates a tight feedback loop: you save a file, and within seconds you see if your fix worked, without running unrelated tests.

## Using Test Templates

## The Repetitive Test Problem

You're adding a new API endpoint to your client. You need to write tests for:
- Success case
- 404 error
- 401 authentication error
- 500 server error
- Network timeout
- Invalid response format

You find yourself copying and pasting test structure repeatedly:

```python
# tests/test_api_client.py
def test_get_user_success(client, mock_session):
    mock_session.get.return_value.json.return_value = {"id": 1}
    result = client.get_user(1)
    assert result["id"] == 1

def test_get_user_not_found(client, mock_session):
    mock_session.get.return_value.status_code = 404
    mock_session.get.return_value.raise_for_status.side_effect = requests.HTTPError()
    with pytest.raises(requests.HTTPError):
        client.get_user(999)

def test_get_user_unauthorized(client, mock_session):
    mock_session.get.return_value.status_code = 401
    mock_session.get.return_value.raise_for_status.side_effect = requests.HTTPError()
    with pytest.raises(requests.HTTPError):
        client.get_user(1)

# Now repeat this pattern for list_users, create_user, update_user, delete_user...
```

This is tedious and error-prone. Each endpoint needs the same test structure, but with different method calls and parameters.

### Iteration 1: Fixture-Based Templates

Create reusable test fixtures that encapsulate common patterns:

```python
# conftest.py
import pytest
import requests
from unittest.mock import Mock

@pytest.fixture
def mock_http_error():
    """Factory fixture for creating HTTP error responses."""
    def _make_error(status_code: int, message: str = None):
        response = Mock()
        response.status_code = status_code
        response.raise_for_status.side_effect = requests.HTTPError(
            message or f"HTTP {status_code}"
        )
        return response
    return _make_error

@pytest.fixture
def mock_success_response():
    """Factory fixture for creating successful responses."""
    def _make_response(data: dict):
        response = Mock()
        response.status_code = 200
        response.json.return_value = data
        response.raise_for_status.return_value = None
        return response
    return _make_response
```

Now use these templates to write tests more concisely:

```python
# tests/test_api_client.py
def test_get_user_success(client, mock_session, mock_success_response):
    mock_session.get.return_value = mock_success_response({"id": 1, "username": "alice"})
    
    result = client.get_user(1)
    
    assert result["id"] == 1
    assert result["username"] == "alice"

def test_get_user_not_found(client, mock_session, mock_http_error):
    mock_session.get.return_value = mock_http_error(404, "User not found")
    
    with pytest.raises(requests.HTTPError, match="404"):
        client.get_user(999)

def test_get_user_unauthorized(client, mock_session, mock_http_error):
    mock_session.get.return_value = mock_http_error(401, "Unauthorized")
    
    with pytest.raises(requests.HTTPError, match="401"):
        client.get_user(1)
```

**Improvement**: Less boilerplate, more readable, consistent error handling.

### Iteration 2: Parameterized Test Templates

For endpoints that share the same error handling patterns, use parametrization:

```python
# tests/test_api_client.py
@pytest.mark.parametrize("status_code,error_message", [
    (400, "Bad Request"),
    (401, "Unauthorized"),
    (403, "Forbidden"),
    (404, "Not Found"),
    (500, "Internal Server Error"),
    (503, "Service Unavailable"),
])
def test_get_user_http_errors(client, mock_session, mock_http_error, status_code, error_message):
    """Test that get_user handles various HTTP errors correctly."""
    mock_session.get.return_value = mock_http_error(status_code, error_message)
    
    with pytest.raises(requests.HTTPError, match=str(status_code)):
        client.get_user(1)
```

**Result**: One test function covers 6 error scenarios. Add a new error code? Just add a line to the parameter list.

### Iteration 3: Class-Based Test Templates

For complex test scenarios that share setup logic, use test classes with fixtures:

```python
# tests/test_api_client.py
class TestAPIEndpoint:
    """Base class template for testing API endpoints."""
    
    @pytest.fixture
    def endpoint_method(self):
        """Override in subclass to specify which method to test."""
        raise NotImplementedError
    
    @pytest.fixture
    def success_response_data(self):
        """Override in subclass to specify expected success data."""
        raise NotImplementedError
    
    @pytest.fixture
    def method_args(self):
        """Override in subclass to specify method arguments."""
        return ()
    
    @pytest.fixture
    def method_kwargs(self):
        """Override in subclass to specify method keyword arguments."""
        return {}
    
    def test_success(self, client, mock_session, mock_success_response, 
                     endpoint_method, success_response_data, method_args, method_kwargs):
        """Test successful endpoint call."""
        mock_session.get.return_value = mock_success_response(success_response_data)
        
        result = endpoint_method(*method_args, **method_kwargs)
        
        assert result == success_response_data
    
    def test_not_found(self, client, mock_session, mock_http_error,
                       endpoint_method, method_args, method_kwargs):
        """Test 404 error handling."""
        mock_session.get.return_value = mock_http_error(404)
        
        with pytest.raises(requests.HTTPError, match="404"):
            endpoint_method(*method_args, **method_kwargs)
    
    def test_unauthorized(self, client, mock_session, mock_http_error,
                          endpoint_method, method_args, method_kwargs):
        """Test 401 error handling."""
        mock_session.get.return_value = mock_http_error(401)
        
        with pytest.raises(requests.HTTPError, match="401"):
            endpoint_method(*method_args, **method_kwargs)

class TestGetUser(TestAPIEndpoint):
    """Test get_user endpoint using template."""
    
    @pytest.fixture
    def endpoint_method(self, client):
        return client.get_user
    
    @pytest.fixture
    def success_response_data(self):
        return {"id": 1, "username": "alice"}
    
    @pytest.fixture
    def method_args(self):
        return (1,)  # user_id

class TestListUsers(TestAPIEndpoint):
    """Test list_users endpoint using template."""
    
    @pytest.fixture
    def endpoint_method(self, client):
        return client.list_users
    
    @pytest.fixture
    def success_response_data(self):
        return [
            {"id": 1, "username": "alice"},
            {"id": 2, "username": "bob"}
        ]
    
    @pytest.fixture
    def method_kwargs(self):
        return {"page": 1, "limit": 10}
```

**Result**: Each endpoint gets comprehensive error handling tests by inheriting from the template. Add a new endpoint? Create a new subclass with 10 lines of code, get 3+ tests automatically.

### Iteration 4: Pytest Plugin for Custom Templates

For organization-wide test patterns, create a pytest plugin:

```python
# pytest_api_templates.py (custom plugin)
import pytest
from typing import Callable, Any, Dict

class APITestTemplate:
    """Template for testing API client methods."""
    
    def __init__(self, method: Callable, success_data: Any, *args, **kwargs):
        self.method = method
        self.success_data = success_data
        self.args = args
        self.kwargs = kwargs
    
    def test_success(self, mock_session, mock_success_response):
        """Test successful API call."""
        mock_session.get.return_value = mock_success_response(self.success_data)
        result = self.method(*self.args, **self.kwargs)
        assert result == self.success_data
    
    def test_http_errors(self, mock_session, mock_http_error):
        """Test HTTP error handling."""
        for status_code in [400, 401, 403, 404, 500, 503]:
            mock_session.get.return_value = mock_http_error(status_code)
            with pytest.raises(Exception):
                self.method(*self.args, **self.kwargs)
    
    def test_network_error(self, mock_session):
        """Test network error handling."""
        mock_session.get.side_effect = requests.exceptions.ConnectionError()
        with pytest.raises(requests.exceptions.ConnectionError):
            self.method(*self.args, **self.kwargs)

@pytest.fixture
def api_test_template():
    """Fixture that returns the template class."""
    return APITestTemplate
```

Use the plugin in your tests:

```python
# tests/test_api_client.py
def test_get_user_suite(client, api_test_template, mock_session, mock_success_response, mock_http_error):
    """Run full test suite for get_user endpoint."""
    template = api_test_template(
        method=client.get_user,
        success_data={"id": 1, "username": "alice"},
        1  # user_id argument
    )
    
    # Run all template tests
    template.test_success(mock_session, mock_success_response)
    template.test_http_errors(mock_session, mock_http_error)
    template.test_network_error(mock_session)
```

### Iteration 5: Code Generation Templates

For truly repetitive patterns, use code generation:

```python
# generate_tests.py
from pathlib import Path

ENDPOINT_TEMPLATE = '''
class Test{endpoint_class}:
    """Test {endpoint_name} endpoint."""
    
    def test_success(self, client, mock_session, mock_success_response):
        mock_session.{http_method}.return_value = mock_success_response({success_data})
        result = client.{endpoint_name}({args})
        assert result == {success_data}
    
    def test_not_found(self, client, mock_session, mock_http_error):
        mock_session.{http_method}.return_value = mock_http_error(404)
        with pytest.raises(requests.HTTPError, match="404"):
            client.{endpoint_name}({args})
    
    def test_unauthorized(self, client, mock_session, mock_http_error):
        mock_session.{http_method}.return_value = mock_http_error(401)
        with pytest.raises(requests.HTTPError, match="401"):
            client.{endpoint_name}({args})
'''

def generate_endpoint_tests(endpoint_name: str, http_method: str, 
                            args: str, success_data: str):
    """Generate test class for an endpoint."""
    endpoint_class = ''.join(word.capitalize() for word in endpoint_name.split('_'))
    
    return ENDPOINT_TEMPLATE.format(
        endpoint_class=endpoint_class,
        endpoint_name=endpoint_name,
        http_method=http_method,
        args=args,
        success_data=success_data
    )

# Generate tests for all endpoints
endpoints = [
    ("get_user", "get", "1", '{"id": 1, "username": "alice"}'),
    ("list_users", "get", "", '[{"id": 1}, {"id": 2}]'),
    ("create_user", "post", '"alice", "alice@example.com"', '{"id": 3}'),
    ("update_user", "patch", '1, username="alice_updated"', '{"id": 1}'),
    ("delete_user", "delete", "1", "None"),
]

output = "# Auto-generated tests\nimport pytest\nimport requests\n\n"
for endpoint_name, http_method, args, success_data in endpoints:
    output += generate_endpoint_tests(endpoint_name, http_method, args, success_data)

Path("tests/test_api_client_generated.py").write_text(output)
print("Generated tests/test_api_client_generated.py")
```

Run the generator:

```bash
$ python generate_tests.py
Generated tests/test_api_client_generated.py
```

**Result**: 15 tests (3 per endpoint × 5 endpoints) generated automatically. Add a new endpoint? Add one line to the `endpoints` list and regenerate.

### When to Use Each Template Approach

| Approach | Best For | Complexity | Flexibility |
|----------|----------|------------|-------------|
| Fixture-based | Common setup patterns | Low | High |
| Parameterized | Multiple similar scenarios | Low | Medium |
| Class-based | Endpoint families | Medium | High |
| Plugin-based | Organization-wide patterns | High | Medium |
| Code generation | Highly repetitive tests | High | Low |

### Decision Framework: Template vs. Manual

**Use templates when**:
- You're writing the same test structure 3+ times
- The pattern is stable and unlikely to change
- New team members need to write similar tests
- You want to enforce consistency across the codebase

**Write manual tests when**:
- The test logic is unique
- The pattern is still evolving
- Flexibility is more important than consistency
- The template would be more complex than the tests themselves

### Pro Tips for Test Templates

**1. Start simple, refactor to templates later**:

Write 3-4 manual tests first. Once you see the pattern, extract it into a template. Don't prematurely abstract.

**2. Document template usage**:

```python
# conftest.py
@pytest.fixture
def api_test_template():
    """
    Template for testing API endpoints.
    
    Usage:
        template = api_test_template(
            method=client.get_user,
            success_data={"id": 1},
            1  # positional args
        )
        template.test_success(mock_session, mock_success_response)
    
    Provides:
        - test_success: Verify successful API call
        - test_http_errors: Test 4xx/5xx error handling
        - test_network_error: Test connection failures
    """
    return APITestTemplate
```

**3. Make templates discoverable**:

Create a `tests/templates/` directory with example usage:

```python
# tests/templates/example_api_endpoint.py
"""
Example of using the API test template.

Copy this file and modify for your endpoint.
"""
import pytest
from conftest import api_test_template

class TestYourEndpoint:
    @pytest.fixture
    def endpoint_method(self, client):
        return client.your_method
    
    @pytest.fixture
    def success_response_data(self):
        return {"your": "data"}
    
    # Tests are inherited from template
```

**4. Version your templates**:

As your API evolves, you might need different template versions:

```python
# conftest.py
@pytest.fixture
def api_test_template_v1():
    """Template for v1 API endpoints."""
    return APITestTemplateV1

@pytest.fixture
def api_test_template_v2():
    """Template for v2 API endpoints (includes auth)."""
    return APITestTemplateV2
```

**5. Combine templates with markers**:

```python
# conftest.py
def pytest_collection_modifyitems(items):
    """Auto-mark tests generated from templates."""
    for item in items:
        if "template" in item.nodeid:
            item.add_marker(pytest.mark.template_generated)
```

Then run or exclude template-generated tests:

```bash
# Run only manually written tests
$ pytest -m "not template_generated"

# Run only template-generated tests
$ pytest -m template_generated
```

## Industry Hacks and Patterns

Professional developers face challenges that textbooks don't cover: legacy code without tests, massive codebases, distributed systems, and non-deterministic behavior. This section reveals battle-tested patterns for these real-world scenarios.

We'll explore these patterns through a realistic scenario: a legacy e-commerce system that's been in production for 5 years, has minimal test coverage, and needs to be modernized without breaking existing functionality.

```python
# src/legacy_order_system.py (the code we inherited)
import datetime
import smtplib
from decimal import Decimal

class OrderProcessor:
    """Legacy order processing system."""
    
    def __init__(self):
        self.db_connection = None
        self.email_server = None
    
    def process_order(self, order_data):
        """
        Process an order. This method does EVERYTHING:
        - Validates order data
        - Checks inventory
        - Calculates pricing
        - Processes payment
        - Updates database
        - Sends confirmation email
        - Updates analytics
        """
        # Validation
        if not order_data.get('customer_id'):
            raise ValueError("Missing customer_id")
        if not order_data.get('items'):
            raise ValueError("No items in order")
        
        # Connect to database (global state!)
        import mysql.connector
        self.db_connection = mysql.connector.connect(
            host="prod-db.company.com",
            user="app_user",
            password="hardcoded_password",  # Yes, really
            database="orders"
        )
        
        # Check inventory (direct DB query)
        cursor = self.db_connection.cursor()
        for item in order_data['items']:
            cursor.execute(
                "SELECT quantity FROM inventory WHERE product_id = %s",
                (item['product_id'],)
            )
            result = cursor.fetchone()
            if not result or result[0] < item['quantity']:
                raise ValueError(f"Insufficient inventory for {item['product_id']}")
        
        # Calculate total (complex business logic)
        total = Decimal('0')
        for item in order_data['items']:
            cursor.execute(
                "SELECT price FROM products WHERE id = %s",
                (item['product_id'],)
            )
            price = Decimal(str(cursor.fetchone()[0]))
            
            # Apply discounts (hardcoded rules)
            if item['quantity'] >= 10:
                price *= Decimal('0.9')  # 10% bulk discount
            if order_data.get('customer_type') == 'premium':
                price *= Decimal('0.95')  # 5% premium discount
            
            total += price * item['quantity']
        
        # Apply tax (varies by state)
        state = order_data.get('shipping_state', 'CA')
        tax_rates = {'CA': 0.0725, 'NY': 0.08, 'TX': 0.0625}
        tax = total * Decimal(str(tax_rates.get(state, 0.06)))
        total += tax
        
        # Process payment (external API call)
        import requests
        payment_response = requests.post(
            "https://payment-gateway.example.com/charge",
            json={
                'amount': float(total),
                'customer_id': order_data['customer_id'],
                'card_token': order_data['payment_token']
            },
            timeout=30
        )
        
        if payment_response.status_code != 200:
            raise Exception("Payment failed")
        
        # Insert order into database
        cursor.execute(
            """INSERT INTO orders 
               (customer_id, total, status, created_at) 
               VALUES (%s, %s, %s, %s)""",
            (order_data['customer_id'], float(total), 'completed', 
             datetime.datetime.now())
        )
        order_id = cursor.lastrowid
        
        # Insert order items
        for item in order_data['items']:
            cursor.execute(
                """INSERT INTO order_items 
                   (order_id, product_id, quantity, price) 
                   VALUES (%s, %s, %s, %s)""",
                (order_id, item['product_id'], item['quantity'], 
                 float(item['price']))
            )
        
        self.db_connection.commit()
        
        # Send confirmation email
        self.email_server = smtplib.SMTP('smtp.company.com', 587)
        self.email_server.starttls()
        self.email_server.login('noreply@company.com', 'email_password')
        
        message = f"""
        Subject: Order Confirmation #{order_id}
        
        Thank you for your order!
        Order ID: {order_id}
        Total: ${total}
        """
        
        self.email_server.sendmail(
            'noreply@company.com',
            order_data['customer_email'],
            message
        )
        
        # Update analytics (another external service)
        requests.post(
            "https://analytics.company.com/event",
            json={
                'event': 'order_completed',
                'order_id': order_id,
                'total': float(total),
                'timestamp': datetime.datetime.now().isoformat()
            }
        )
        
        # Cleanup
        cursor.close()
        self.db_connection.close()
        self.email_server.quit()
        
        return order_id
```

This is the reality of legacy code: a 150-line method that does everything, has no tests, uses global state, makes external calls, and contains critical business logic. How do you test this without rewriting it?

## The Legacy Code Dilemma

You need to add a feature to `process_order()`: support for promotional codes. But first, you need tests to ensure you don't break existing functionality. The problem: this code is untestable in its current form.

**The naive approach** (doesn't work):

```python
# tests/test_order_processor.py (naive attempt)
import pytest
from src.legacy_order_system import OrderProcessor

def test_process_order():
    processor = OrderProcessor()
    
    order_data = {
        'customer_id': 123,
        'customer_email': 'test@example.com',
        'items': [
            {'product_id': 1, 'quantity': 2, 'price': 10.00}
        ],
        'payment_token': 'tok_test'
    }
    
    order_id = processor.process_order(order_data)
    
    assert order_id > 0
```

Run this test:

```bash
$ pytest tests/test_order_processor.py::test_process_order -v
```

**Output**:

```text
tests/test_order_processor.py::test_process_order FAILED

E   mysql.connector.errors.InterfaceError: 2003: Can't connect to MySQL server on 'prod-db.company.com'

======================== 1 failed in 2.34s ========================
```

**Root cause**: The test tries to connect to the production database. Even if we had a test database, the test would still:
- Make real HTTP requests to payment gateway
- Send real emails
- Depend on database state
- Take 30+ seconds to run

### Iteration 1: The Characterization Test Pattern

Before refactoring, we need to understand what the code currently does. Create "characterization tests" that document existing behavior:

```python
# tests/test_order_processor_characterization.py
import pytest
from unittest.mock import Mock, patch, MagicMock
from src.legacy_order_system import OrderProcessor
from decimal import Decimal

@pytest.fixture
def mock_database():
    """Mock the database connection."""
    with patch('mysql.connector.connect') as mock_connect:
        mock_conn = MagicMock()
        mock_cursor = MagicMock()
        
        # Setup cursor to return expected data
        mock_cursor.fetchone.side_effect = [
            (100,),  # inventory quantity for first item
            (Decimal('10.00'),),  # price for first item
        ]
        mock_cursor.lastrowid = 12345  # order_id after insert
        
        mock_conn.cursor.return_value = mock_cursor
        mock_connect.return_value = mock_conn
        
        yield mock_conn, mock_cursor

@pytest.fixture
def mock_payment():
    """Mock the payment gateway."""
    with patch('requests.post') as mock_post:
        mock_response = Mock()
        mock_response.status_code = 200
        mock_post.return_value = mock_response
        yield mock_post

@pytest.fixture
def mock_email():
    """Mock the email server."""
    with patch('smtplib.SMTP') as mock_smtp:
        mock_server = MagicMock()
        mock_smtp.return_value = mock_server
        yield mock_server

def test_process_order_basic_flow(mock_database, mock_payment, mock_email):
    """
    Characterization test: Document what process_order currently does.
    
    This test doesn't verify correctness—it verifies current behavior.
    Once we have this safety net, we can refactor.
    """
    mock_conn, mock_cursor = mock_database
    processor = OrderProcessor()
    
    order_data = {
        'customer_id': 123,
        'customer_email': 'test@example.com',
        'items': [
            {'product_id': 1, 'quantity': 2, 'price': 10.00}
        ],
        'payment_token': 'tok_test',
        'shipping_state': 'CA'
    }
    
    order_id = processor.process_order(order_data)
    
    # Verify the order ID was returned
    assert order_id == 12345
    
    # Verify database interactions (characterize current behavior)
    assert mock_cursor.execute.call_count == 4  # inventory, price, insert order, insert items
    
    # Verify payment was processed
    mock_payment.assert_called_once()
    payment_call = mock_payment.call_args
    assert payment_call[1]['json']['amount'] == pytest.approx(21.45, rel=0.01)  # 20 + 7.25% tax
    
    # Verify email was sent
    mock_email.sendmail.assert_called_once()
    email_call = mock_email.sendmail.call_args
    assert 'Order Confirmation #12345' in email_call[0][2]
```

Run the characterization test:

```bash
$ pytest tests/test_order_processor_characterization.py -v
```

**Output**:

```text
tests/test_order_processor_characterization.py::test_process_order_basic_flow PASSED

======================== 1 passed in 0.23s ========================
```

**Success!** We've created a test that runs without external dependencies and documents current behavior.

### Iteration 2: The Seam Pattern

Now we can safely add features. We need to test promotional codes without refactoring the entire method. Use "seams"—points where we can inject test behavior.

**Add a seam for discount calculation**:

```python
# src/legacy_order_system.py (minimal change)
class OrderProcessor:
    def __init__(self):
        self.db_connection = None
        self.email_server = None
        self._discount_calculator = None  # Seam for testing
    
    def _calculate_discount(self, item, order_data):
        """
        Seam: Extract discount logic so it can be overridden in tests.
        """
        if self._discount_calculator:
            return self._discount_calculator(item, order_data)
        
        # Original logic (unchanged)
        discount = Decimal('1.0')
        if item['quantity'] >= 10:
            discount *= Decimal('0.9')
        if order_data.get('customer_type') == 'premium':
            discount *= Decimal('0.95')
        return discount
    
    def process_order(self, order_data):
        # ... (earlier code unchanged)
        
        # Calculate total (now uses seam)
        total = Decimal('0')
        for item in order_data['items']:
            cursor.execute(
                "SELECT price FROM products WHERE id = %s",
                (item['product_id'],)
            )
            price = Decimal(str(cursor.fetchone()[0]))
            
            # Use the seam
            discount = self._calculate_discount(item, order_data)
            price *= discount
            
            total += price * item['quantity']
        
        # ... (rest unchanged)
```

Now test the new promotional code feature:

```python
# tests/test_order_processor_promo_codes.py
def test_promotional_code_applies_discount(mock_database, mock_payment, mock_email):
    """Test that promotional codes reduce the order total."""
    mock_conn, mock_cursor = mock_database
    processor = OrderProcessor()
    
    # Inject test behavior through the seam
    def promo_discount_calculator(item, order_data):
        base_discount = Decimal('1.0')
        if item['quantity'] >= 10:
            base_discount *= Decimal('0.9')
        if order_data.get('customer_type') == 'premium':
            base_discount *= Decimal('0.95')
        
        # NEW: Apply promo code discount
        if order_data.get('promo_code') == 'SAVE20':
            base_discount *= Decimal('0.8')  # 20% off
        
        return base_discount
    
    processor._discount_calculator = promo_discount_calculator
    
    order_data = {
        'customer_id': 123,
        'customer_email': 'test@example.com',
        'items': [
            {'product_id': 1, 'quantity': 2, 'price': 10.00}
        ],
        'payment_token': 'tok_test',
        'shipping_state': 'CA',
        'promo_code': 'SAVE20'  # NEW
    }
    
    order_id = processor.process_order(order_data)
    
    # Verify discount was applied
    payment_call = mock_payment.call_args
    # Original: 20 + 7.25% tax = 21.45
    # With 20% promo: 16 + 7.25% tax = 17.16
    assert payment_call[1]['json']['amount'] == pytest.approx(17.16, rel=0.01)
```

**Result**: We tested the new feature without rewriting the legacy code. The seam allows us to inject test behavior while preserving production behavior.

### Iteration 3: The Approval Testing Pattern

For complex legacy code where you don't fully understand the behavior, use "approval testing" (also called "golden master testing"):

```python
# tests/test_order_processor_approval.py
import pytest
import json
from pathlib import Path
from unittest.mock import Mock, patch, MagicMock
from src.legacy_order_system import OrderProcessor

@pytest.fixture
def approval_test_setup(mock_database, mock_payment, mock_email):
    """Setup for approval testing."""
    mock_conn, mock_cursor = mock_database
    
    # Capture all interactions
    interactions = {
        'database_queries': [],
        'payment_calls': [],
        'email_calls': []
    }
    
    # Wrap mocks to capture calls
    original_execute = mock_cursor.execute
    def capture_execute(query, params=None):
        interactions['database_queries'].append({
            'query': query,
            'params': params
        })
        return original_execute(query, params)
    mock_cursor.execute = capture_execute
    
    original_payment = mock_payment
    def capture_payment(*args, **kwargs):
        interactions['payment_calls'].append({
            'args': args,
            'kwargs': kwargs
        })
        return original_payment(*args, **kwargs)
    mock_payment.side_effect = capture_payment
    
    original_sendmail = mock_email.sendmail
    def capture_email(*args, **kwargs):
        interactions['email_calls'].append({
            'args': args,
            'kwargs': kwargs
        })
        return original_sendmail(*args, **kwargs)
    mock_email.sendmail = capture_email
    
    return interactions

def test_order_processing_approval(approval_test_setup):
    """
    Approval test: Capture complete behavior and compare to approved baseline.
    
    First run: Manually review output and approve it.
    Subsequent runs: Verify behavior hasn't changed.
    """
    interactions = approval_test_setup
    processor = OrderProcessor()
    
    order_data = {
        'customer_id': 123,
        'customer_email': 'test@example.com',
        'items': [
            {'product_id': 1, 'quantity': 2, 'price': 10.00}
        ],
        'payment_token': 'tok_test',
        'shipping_state': 'CA'
    }
    
    order_id = processor.process_order(order_data)
    
    # Serialize all interactions
    result = {
        'order_id': order_id,
        'database_queries': interactions['database_queries'],
        'payment_calls': [
            {
                'url': call['args'][0] if call['args'] else None,
                'amount': call['kwargs']['json']['amount']
            }
            for call in interactions['payment_calls']
        ],
        'email_calls': [
            {
                'to': call['args'][1],
                'subject_line': call['args'][2].split('\n')[1]
            }
            for call in interactions['email_calls']
        ]
    }
    
    # Compare to approved baseline
    approval_file = Path('tests/approvals/order_processing.approved.json')
    
    if not approval_file.exists():
        # First run: Save as baseline
        approval_file.parent.mkdir(exist_ok=True)
        approval_file.write_text(json.dumps(result, indent=2))
        pytest.skip("Baseline created. Review and approve.")
    
    # Compare to baseline
    approved = json.loads(approval_file.read_text())
    assert result == approved, "Behavior changed! Review differences."
```

**First run** (creates baseline):

```bash
$ pytest tests/test_order_processor_approval.py -v
```

**Output**:

```text
tests/test_order_processor_approval.py::test_order_processing_approval SKIPPED (Baseline created. Review and approve.)
```

Review the generated file:

```json
{
  "order_id": 12345,
  "database_queries": [
    {
      "query": "SELECT quantity FROM inventory WHERE product_id = %s",
      "params": [1]
    },
    {
      "query": "SELECT price FROM products WHERE id = %s",
      "params": [1]
    },
    {
      "query": "INSERT INTO orders (customer_id, total, status, created_at) VALUES (%s, %s, %s, %s)",
      "params": [123, 21.45, "completed", "2024-01-15T10:30:00"]
    }
  ],
  "payment_calls": [
    {
      "url": "https://payment-gateway.example.com/charge",
      "amount": 21.45
    }
  ],
  "email_calls": [
    {
      "to": "test@example.com",
      "subject_line": "Subject: Order Confirmation #12345"
    }
  ]
}
```

If this looks correct, approve it. **Subsequent runs** verify nothing changed:

```bash
$ pytest tests/test_order_processor_approval.py -v
```

**Output**:

```text
tests/test_order_processor_approval.py::test_order_processing_approval PASSED
```

If behavior changes (intentionally or not), the test fails and shows the diff.

### When to Use Each Pattern

| Pattern | Best For | Effort | Safety |
|---------|----------|--------|--------|
| Characterization | Understanding existing behavior | Low | Medium |
| Seam | Adding features to legacy code | Medium | High |
| Approval | Complex behavior you don't fully understand | High | Very High |

### Pro Tips for Legacy Code Testing

**1. Start with the happy path**:

Don't try to test every edge case immediately. Get one end-to-end test working first, then expand coverage.

**2. Use pytest's monkeypatch for temporary seams**:

```python
def test_with_temporary_seam(monkeypatch):
    """Use monkeypatch to inject test behavior without modifying code."""
    processor = OrderProcessor()
    
    # Temporarily replace a method
    def mock_calculate_discount(item, order_data):
        return Decimal('0.8')
    
    monkeypatch.setattr(processor, '_calculate_discount', mock_calculate_discount)
    
    # Test proceeds with mocked behavior
```

**3. Document what you're NOT testing**:

```python
def test_process_order_basic_flow(mock_database, mock_payment, mock_email):
    """
    Test basic order processing flow.
    
    NOT TESTED (yet):
    - Invalid payment tokens
    - Database connection failures
    - Email delivery failures
    - Inventory edge cases (negative quantities, etc.)
    - Tax calculation for international orders
    
    These will be added as we refactor.
    """
    pass
```

**4. Use coverage to find untested paths**:

```bash
$ pytest tests/test_order_processor_characterization.py --cov=src.legacy_order_system --cov-report=html
```

Open `htmlcov/index.html` to see which lines aren't covered. Prioritize testing the most critical paths first.

**5. Refactor incrementally with test coverage**:

Once you have characterization tests:
1. Extract one small piece of logic
2. Write focused tests for it
3. Replace the original code with a call to the extracted function
4. Verify characterization tests still pass
5. Repeat

This is the "Strangler Fig" pattern—gradually replace legacy code while maintaining functionality.

## Incremental Testing for Large Projects

## The Large Codebase Problem

Your company has a monolithic application with 500,000 lines of code and 2,000 test files. Running the full test suite takes 45 minutes. You're working on a small feature in the authentication module. How do you get fast feedback without waiting for the entire suite?

**The naive approach** (doesn't scale):

```bash
# Run everything every time
$ pytest tests/
```

**Output**:

```text
======================== 12,847 passed in 2734.23s (45 minutes) ========================
```

This is untenable for rapid development. You need incremental testing strategies.

### Iteration 1: Test Impact Analysis

Only run tests affected by your changes. First, understand the dependency graph:

```python
# tools/test_impact_analyzer.py
import ast
import sys
from pathlib import Path
from typing import Set, Dict, List

class DependencyAnalyzer(ast.NodeVisitor):
    """Analyze Python files to extract import dependencies."""
    
    def __init__(self):
        self.imports: Set[str] = set()
    
    def visit_Import(self, node):
        for alias in node.names:
            self.imports.add(alias.name.split('.')[0])
    
    def visit_ImportFrom(self, node):
        if node.module:
            self.imports.add(node.module.split('.')[0])

def get_dependencies(file_path: Path) -> Set[str]:
    """Extract all imports from a Python file."""
    try:
        tree = ast.parse(file_path.read_text())
        analyzer = DependencyAnalyzer()
        analyzer.visit(tree)
        return analyzer.imports
    except:
        return set()

def build_dependency_graph(src_dir: Path) -> Dict[str, Set[str]]:
    """Build a graph of which modules depend on which."""
    graph = {}
    
    for py_file in src_dir.rglob('*.py'):
        module_name = str(py_file.relative_to(src_dir)).replace('/', '.').replace('.py', '')
        graph[module_name] = get_dependencies(py_file)
    
    return graph

def find_affected_tests(changed_files: List[str], src_dir: Path, test_dir: Path) -> Set[str]:
    """
    Find all tests that might be affected by changes to given files.
    
    Strategy:
    1. Build dependency graph
    2. Find all modules that import changed modules (direct dependencies)
    3. Find all modules that import those modules (transitive dependencies)
    4. Find test files that test any affected modules
    """
    graph = build_dependency_graph(src_dir)
    
    # Convert changed files to module names
    changed_modules = set()
    for file_path in changed_files:
        if file_path.startswith('src/'):
            module = file_path.replace('src/', '').replace('/', '.').replace('.py', '')
            changed_modules.add(module)
    
    # Find all modules affected by changes (transitive closure)
    affected = changed_modules.copy()
    changed = True
    while changed:
        changed = False
        for module, deps in graph.items():
            if module not in affected and affected & deps:
                affected.add(module)
                changed = True
    
    # Find test files for affected modules
    affected_tests = set()
    for test_file in test_dir.rglob('test_*.py'):
        test_deps = get_dependencies(test_file)
        if affected & test_deps:
            affected_tests.add(str(test_file))
    
    return affected_tests

if __name__ == '__main__':
    # Get changed files from git
    import subprocess
    result = subprocess.run(
        ['git', 'diff', '--name-only', 'HEAD'],
        capture_output=True,
        text=True
    )
    changed_files = result.stdout.strip().split('\n')
    
    affected_tests = find_affected_tests(
        changed_files,
        Path('src'),
        Path('tests')
    )
    
    print(f"Changed files: {len(changed_files)}")
    print(f"Affected tests: {len(affected_tests)}")
    print("\nRun these tests:")
    for test in sorted(affected_tests):
        print(f"  {test}")
```

Use the analyzer:

```bash
$ python tools/test_impact_analyzer.py
```

**Output**:

```text
Changed files: 3
Affected tests: 47

Run these tests:
  tests/test_auth.py
  tests/test_user_management.py
  tests/test_session_handling.py
  ...
```

Now run only affected tests:

```bash
$ pytest $(python tools/test_impact_analyzer.py | grep "tests/" | tr '\n' ' ')
```

**Output**:

```text
======================== 47 passed in 12.34s ========================
```

**Result**: 45 minutes → 12 seconds (220x speedup)

### Iteration 2: Layered Testing Strategy

Organize tests by speed and scope:

```python
# conftest.py
import pytest

def pytest_configure(config):
    config.addinivalue_line("markers", "unit: fast unit tests (< 0.1s each)")
    config.addinivalue_line("markers", "integration: integration tests (< 1s each)")
    config.addinivalue_line("markers", "e2e: end-to-end tests (< 10s each)")
    config.addinivalue_line("markers", "slow: slow tests (> 10s each)")
```

Mark your tests:

```python
# tests/test_auth.py
import pytest

@pytest.mark.unit
def test_password_hashing():
    """Fast unit test."""
    pass

@pytest.mark.integration
def test_login_with_database():
    """Integration test with database."""
    pass

@pytest.mark.e2e
def test_full_authentication_flow():
    """End-to-end test through web interface."""
    pass

@pytest.mark.slow
def test_password_reset_email_delivery():
    """Slow test that sends real email."""
    pass
```

Create a testing pyramid:

```bash
# Level 1: Unit tests (run constantly during development)
$ pytest -m unit

# Level 2: Integration tests (run before commit)
$ pytest -m "unit or integration"

# Level 3: E2E tests (run in CI before merge)
$ pytest -m "unit or integration or e2e"

# Level 4: Full suite (run nightly)
$ pytest
```

**Typical results**:

| Level | Tests | Time | When to Run |
|-------|-------|------|-------------|
| Unit | 8,000 | 2 min | Every save (with pytest-watch) |
| Integration | 3,500 | 8 min | Before commit |
| E2E | 1,200 | 25 min | Before merge (CI) |
| Full | 12,847 | 45 min | Nightly (CI) |

### Iteration 3: Parallel Test Execution with Smart Scheduling

Use pytest-xdist with custom scheduling to optimize parallel execution:

```python
# conftest.py
import pytest

def pytest_collection_modifyitems(session, config, items):
    """
    Reorder tests for optimal parallel execution.
    
    Strategy:
    1. Run fast tests first (quick feedback)
    2. Distribute slow tests evenly across workers
    3. Group tests by fixture scope (reduce setup/teardown)
    """
    # Separate tests by speed
    fast_tests = []
    medium_tests = []
    slow_tests = []
    
    for item in items:
        # Estimate test duration from markers or historical data
        if item.get_closest_marker('unit'):
            fast_tests.append(item)
        elif item.get_closest_marker('integration'):
            medium_tests.append(item)
        else:
            slow_tests.append(item)
    
    # Reorder: fast first, then interleave medium and slow
    items[:] = fast_tests + medium_tests + slow_tests
```

Run with optimal parallelization:

```bash
# Use all CPU cores, with smart scheduling
$ pytest -n auto --dist loadscope
```

**Result**: 45 minutes → 13 minutes (3.5x speedup on 4-core machine)

### Iteration 4: Test Sharding for CI

In CI, run tests across multiple machines:

```yaml
# .github/workflows/test.yml
name: Test Suite

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard: [1, 2, 3, 4, 5, 6, 7, 8]
    steps:
      - uses: actions/checkout@v2
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: 3.11
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-xdist pytest-split
      - name: Run tests (shard ${{ matrix.shard }}/8)
        run: |
          pytest --splits 8 --group ${{ matrix.shard }} --durations-path .test-durations
```

**Result**: 45 minutes → 6 minutes (8 machines running in parallel)

### Iteration 5: Caching Test Results

Cache test results to skip tests that haven't changed:

```python
# conftest.py
import pytest
import hashlib
import json
from pathlib import Path

def pytest_collection_modifyitems(session, config, items):
    """Skip tests whose code and dependencies haven't changed."""
    cache_file = Path('.pytest_cache/test_hashes.json')
    
    # Load previous hashes
    if cache_file.exists():
        previous_hashes = json.loads(cache_file.read_text())
    else:
        previous_hashes = {}
    
    current_hashes = {}
    
    for item in items:
        # Compute hash of test code + dependencies
        test_file = Path(item.fspath)
        test_code = test_file.read_text()
        
        # Include imported modules in hash
        dependencies = get_dependencies(test_file)
        dep_code = ''.join(
            Path(dep).read_text() 
            for dep in dependencies 
            if Path(dep).exists()
        )
        
        combined = test_code + dep_code
        test_hash = hashlib.sha256(combined.encode()).hexdigest()
        
        current_hashes[item.nodeid] = test_hash
        
        # Skip if unchanged and previously passed
        if (item.nodeid in previous_hashes and 
            previous_hashes[item.nodeid] == test_hash):
            item.add_marker(pytest.mark.skip(reason="unchanged since last run"))
    
    # Save current hashes
    cache_file.parent.mkdir(exist_ok=True)
    cache_file.write_text(json.dumps(current_hashes, indent=2))
```

**First run** (no cache):

```bash
$ pytest tests/
```

**Output**:

```text
======================== 12,847 passed in 2734.23s ========================
```

**Second run** (with cache, no changes):

```bash
$ pytest tests/
```

**Output**:

```text
======================== 12,847 skipped in 23.45s ========================
```

**After making changes** (only affected tests run):

```bash
$ pytest tests/
```

**Output**:

```text
======================== 47 passed, 12,800 skipped in 34.56s ========================
```

### Iteration 6: Focused Test Suites

Create focused test suites for common workflows:

```ini
# pytest.ini
[pytest]
markers =
    smoke: critical path tests (must pass before any commit)
    auth: authentication and authorization tests
    payment: payment processing tests
    api: API endpoint tests
    ui: user interface tests

# Define test suites
[pytest:smoke]
testpaths = tests/smoke
markers = smoke

[pytest:auth]
testpaths = tests/auth tests/user_management
markers = auth

[pytest:payment]
testpaths = tests/payment tests/billing
markers = payment
```

Run focused suites:

```bash
# Smoke test (2 minutes)
$ pytest -c pytest.ini::smoke

# Auth suite (5 minutes)
$ pytest -c pytest.ini::auth

# Payment suite (8 minutes)
$ pytest -c pytest.ini::payment
```

### Decision Framework: Which Strategy When?

| Scenario | Strategy | Speedup | Complexity |
|----------|----------|---------|------------|
| Local development | Test impact analysis | 100-1000x | Medium |
| Pre-commit | Layered testing (unit + integration) | 5-10x | Low |
| CI pull request | Parallel execution + sharding | 4-8x | Medium |
| Nightly CI | Full suite with caching | 2-3x | Low |
| Debugging | Single test with --lf | ∞ | Low |

### Pro Tips for Large Codebases

**1. Measure test duration and optimize slowest tests**:

```bash
# Find slowest tests
$ pytest --durations=20
```

**Output**:

```text
======================== slowest 20 durations ========================
45.23s call     tests/test_payment.py::test_full_payment_flow
32.11s call     tests/test_email.py::test_send_bulk_emails
28.45s call     tests/test_report.py::test_generate_annual_report
...
```

Focus optimization efforts on these tests first.

**2. Use pytest-timeout to prevent hanging tests**:

```bash
$ pip install pytest-timeout
$ pytest --timeout=10  # Fail any test that takes > 10 seconds
```

**3. Create a "quick check" command**:

```bash
# Add to Makefile or package.json scripts
quick-test:
    pytest -m "unit and not slow" -x --ff
```

**4. Use pytest-monitor to track test performance over time**:

```bash
$ pip install pytest-monitor
$ pytest --monitor  # Stores metrics in SQLite database
```

Then analyze trends:

```python
# tools/analyze_test_performance.py
import sqlite3
import pandas as pd

conn = sqlite3.connect('.pymon')
df = pd.read_sql_query("""
    SELECT 
        item_path,
        AVG(item_total_time) as avg_time,
        COUNT(*) as run_count
    FROM TEST_METRICS
    GROUP BY item_path
    ORDER BY avg_time DESC
    LIMIT 20
""", conn)

print("Tests getting slower over time:")
print(df)
```

**5. Document your testing strategy**:

## Contract Testing for Microservices

## The Microservices Testing Problem

Your application is split into microservices:
- **User Service**: Manages user accounts
- **Order Service**: Processes orders
- **Payment Service**: Handles payments
- **Notification Service**: Sends emails/SMS

Each service has its own test suite, but integration failures still occur in production. The problem: services evolve independently, and their contracts (APIs) drift.

**Example failure scenario**:

Order Service expects Payment Service to return:

```json
{
  "transaction_id": "txn_123",
  "status": "completed",
  "amount": 99.99
}
```

But Payment Service was updated to return:

```json
{
  "id": "txn_123",
  "state": "success",
  "total": 99.99
}
```

Field names changed. Order Service breaks in production. How do you catch this before deployment?

### Iteration 1: Consumer-Driven Contract Testing

The consumer (Order Service) defines what it expects from the provider (Payment Service):

```python
# order_service/tests/contracts/test_payment_service_contract.py
import pytest
import requests
from pact import Consumer, Provider, Like, EachLike

# Define the contract
pact = Consumer('OrderService').has_pact_with(Provider('PaymentService'))

@pytest.fixture(scope='module')
def payment_service_contract():
    pact.start_service()
    yield pact
    pact.stop_service()

def test_process_payment_contract(payment_service_contract):
    """
    Contract: Order Service expects Payment Service to process payments.
    
    This test defines what Order Service needs from Payment Service.
    """
    expected_response = {
        'transaction_id': Like('txn_123'),  # String, any value
        'status': Like('completed'),         # String, any value
        'amount': Like(99.99)                # Number, any value
    }
    
    (pact
     .given('a valid payment request')
     .upon_receiving('a request to process payment')
     .with_request(
         method='POST',
         path='/api/v1/payments',
         headers={'Content-Type': 'application/json'},
         body={
             'order_id': Like('ord_123'),
             'amount': Like(99.99),
             'payment_method': Like('credit_card')
         }
     )
     .will_respond_with(200, body=expected_response))
    
    with pact:
        # Make actual request to mock Payment Service
        response = requests.post(
            f'{pact.uri}/api/v1/payments',
            json={
                'order_id': 'ord_456',
                'amount': 149.99,
                'payment_method': 'credit_card'
            }
        )
        
        assert response.status_code == 200
        data = response.json()
        assert 'transaction_id' in data
        assert 'status' in data
        assert 'amount' in data
```

Run the contract test:

```bash
$ pytest order_service/tests/contracts/test_payment_service_contract.py -v
```

**Output**:

```text
order_service/tests/contracts/test_payment_service_contract.py::test_process_payment_contract PASSED

Pact file written to: pacts/orderservice-paymentservice.json
```

This generates a contract file:

```json
{
  "consumer": {
    "name": "OrderService"
  },
  "provider": {
    "name": "PaymentService"
  },
  "interactions": [
    {
      "description": "a request to process payment",
      "providerState": "a valid payment request",
      "request": {
        "method": "POST",
        "path": "/api/v1/payments",
        "headers": {
          "Content-Type": "application/json"
        },
        "body": {
          "order_id": "ord_123",
          "amount": 99.99,
          "payment_method": "credit_card"
        }
      },
      "response": {
        "status": 200,
        "body": {
          "transaction_id": "txn_123",
          "status": "completed",
          "amount": 99.99
        }
      }
    }
  ]
}
```

### Iteration 2: Provider Verification

Payment Service must verify it satisfies the contract:

```python
# payment_service/tests/contracts/test_contract_verification.py
import pytest
from pact import Verifier

def test_payment_service_honors_contracts():
    """
    Verify that Payment Service satisfies all consumer contracts.
    
    This test runs against the actual Payment Service implementation.
    """
    verifier = Verifier(
        provider='PaymentService',
        provider_base_url='http://localhost:8001'  # Payment Service URL
    )
    
    # Verify against all consumer contracts
    output, logs = verifier.verify_pacts(
        './pacts/orderservice-paymentservice.json',
        provider_states_setup_url='http://localhost:8001/_pact/provider_states'
    )
    
    assert output == 0, f"Contract verification failed:\n{logs}"
```

**If Payment Service changes its response format**, the verification fails:

```bash
$ pytest payment_service/tests/contracts/test_contract_verification.py -v
```

**Output**:

```text
payment_service/tests/contracts/test_contract_verification.py::test_payment_service_honors_contracts FAILED

E   AssertionError: Contract verification failed:
E   
E   Verifying a pact between OrderService and PaymentService
E     a request to process payment
E       with POST /api/v1/payments
E         returns a response which
E           has status code 200 (FAILED)
E           has a matching body (FAILED)
E   
E   Failures:
E   
E   1) Verifying a pact between OrderService and PaymentService - a request to process payment
E      1.1) has a matching body
E           Expected field 'transaction_id' but got 'id'
E           Expected field 'status' but got 'state'
E           Expected field 'amount' but got 'total'
```

**Root cause identified**: Payment Service changed its response format without updating the contract. This would have caused production failures, but we caught it in testing.

### Iteration 3: Contract Testing Workflow

**Step 1: Consumer defines contract** (Order Service):

```python
# order_service/tests/contracts/test_payment_service_contract.py
def test_process_payment_contract(payment_service_contract):
    """Define what Order Service needs from Payment Service."""
    # ... (contract definition from Iteration 1)
```

**Step 2: Consumer publishes contract** to a Pact Broker (central repository):

```bash
# In Order Service CI pipeline
$ pact-broker publish pacts/ \
    --consumer-app-version=$GIT_COMMIT \
    --broker-base-url=https://pact-broker.company.com \
    --broker-token=$PACT_BROKER_TOKEN
```

**Step 3: Provider verifies contract** (Payment Service):

```python
# payment_service/tests/contracts/test_contract_verification.py
def test_payment_service_honors_contracts():
    """Verify Payment Service satisfies all consumer contracts."""
    verifier = Verifier(
        provider='PaymentService',
        provider_base_url='http://localhost:8001'
    )
    
    # Fetch contracts from Pact Broker
    output, logs = verifier.verify_with_broker(
        broker_url='https://pact-broker.company.com',
        broker_token=os.environ['PACT_BROKER_TOKEN'],
        provider_version=os.environ['GIT_COMMIT'],
        publish_verification_results=True
    )
    
    assert output == 0, f"Contract verification failed:\n{logs}"
```

**Step 4: Can-I-Deploy check** before deployment:

```bash
# In Payment Service CI pipeline, before deploying
$ pact-broker can-i-deploy \
    --pacticipant PaymentService \
    --version $GIT_COMMIT \
    --to production \
    --broker-base-url https://pact-broker.company.com \
    --broker-token $PACT_BROKER_TOKEN
```

**Output if contracts are satisfied**:

```text
Computer says yes \o/

PaymentService version abc123 can be deployed to production
All required contracts are verified
```

**Output if contracts are broken**:

```text
Computer says no ¯\_(ツ)_/¯

PaymentService version abc123 cannot be deployed to production

Reason: OrderService requires PaymentService to satisfy contract version xyz789
        but verification failed

Details: https://pact-broker.company.com/pacts/...
```

### Iteration 4: Handling Contract Evolution

Contracts need to evolve. Use versioning and backward compatibility:

```python
# payment_service/tests/contracts/test_contract_verification.py
def test_payment_service_honors_contracts_v1():
    """Verify Payment Service satisfies v1 contracts (legacy)."""
    verifier = Verifier(
        provider='PaymentService',
        provider_base_url='http://localhost:8001'
    )
    
    output, logs = verifier.verify_with_broker(
        broker_url='https://pact-broker.company.com',
        broker_token=os.environ['PACT_BROKER_TOKEN'],
        provider_version=os.environ['GIT_COMMIT'],
        consumer_version_selectors=[
            {'tag': 'v1', 'latest': True}  # Only verify v1 contracts
        ]
    )
    
    assert output == 0

def test_payment_service_honors_contracts_v2():
    """Verify Payment Service satisfies v2 contracts (current)."""
    verifier = Verifier(
        provider='PaymentService',
        provider_base_url='http://localhost:8001'
    )
    
    output, logs = verifier.verify_with_broker(
        broker_url='https://pact-broker.company.com',
        broker_token=os.environ['PACT_BROKER_TOKEN'],
        provider_version=os.environ['GIT_COMMIT'],
        consumer_version_selectors=[
            {'tag': 'v2', 'latest': True}  # Only verify v2 contracts
        ]
    )
    
    assert output == 0
```

**Migration strategy**:

1. Payment Service supports both v1 and v2 response formats
2. Consumers gradually migrate from v1 to v2
3. Once all consumers are on v2, remove v1 support
4. Contract tests ensure no consumer is left behind

### Iteration 5: Testing Multiple Consumers

Payment Service has multiple consumers. Test all contracts:

```python
# payment_service/tests/contracts/test_all_consumer_contracts.py
import pytest
from pact import Verifier

@pytest.mark.parametrize('consumer', [
    'OrderService',
    'SubscriptionService',
    'RefundService',
    'AdminDashboard'
])
def test_payment_service_honors_consumer_contract(consumer):
    """Verify Payment Service satisfies contract with each consumer."""
    verifier = Verifier(
        provider='PaymentService',
        provider_base_url='http://localhost:8001'
    )
    
    output, logs = verifier.verify_with_broker(
        broker_url='https://pact-broker.company.com',
        broker_token=os.environ['PACT_BROKER_TOKEN'],
        provider_version=os.environ['GIT_COMMIT'],
        consumer_version_selectors=[
            {'consumer': consumer, 'latest': True}
        ]
    )
    
    assert output == 0, f"Contract with {consumer} failed:\n{logs}"
```

Run all consumer contract tests:

```bash
$ pytest payment_service/tests/contracts/test_all_consumer_contracts.py -v
```

**Output**:

```text
test_all_consumer_contracts.py::test_payment_service_honors_consumer_contract[OrderService] PASSED
test_all_consumer_contracts.py::test_payment_service_honors_consumer_contract[SubscriptionService] PASSED
test_all_consumer_contracts.py::test_payment_service_honors_consumer_contract[RefundService] FAILED
test_all_consumer_contracts.py::test_payment_service_honors_consumer_contract[AdminDashboard] PASSED

E   AssertionError: Contract with RefundService failed:
E   Expected field 'refund_id' but got 'transaction_id'
```

**Diagnostic**: RefundService expects a different response format. We need to either:
1. Update Payment Service to support RefundService's expectations
2. Update RefundService to match Payment Service's format
3. Version the API to support both

### When to Use Contract Testing

**Optimal scenarios**:
- Microservices architecture with multiple teams
- Services deployed independently
- API contracts that evolve over time
- Need to prevent integration failures in production

**When to avoid**:
- Monolithic applications (use integration tests instead)
- Services that are always deployed together
- Rapidly changing APIs in early development (contracts add overhead)
- Internal services with a single consumer (direct integration tests are simpler)

### Pro Tips for Contract Testing

**1. Start with critical paths**:

Don't contract-test every endpoint. Focus on:
- Payment processing
- Authentication
- Data synchronization
- Critical business workflows

**2. Use provider states for setup**:

```python
# payment_service/app.py
@app.route('/_pact/provider_states', methods=['POST'])
def provider_states():
    """Setup provider state for contract testing."""
    state = request.json['state']
    
    if state == 'a valid payment request':
        # Setup: Ensure test data exists
        setup_test_payment_data()
    elif state == 'insufficient funds':
        # Setup: Configure account with low balance
        setup_insufficient_funds_scenario()
    
    return jsonify({'result': 'success'})
```

**3. Use contract testing alongside integration tests**:

Contract tests verify the interface. Integration tests verify the behavior:

```python
# order_service/tests/integration/test_payment_integration.py
def test_full_payment_flow_integration():
    """
    Integration test: Verify complete payment flow.
    
    This complements contract tests by testing actual behavior,
    not just the interface.
    """
    # Create order
    order = create_test_order()
    
    # Process payment (calls real Payment Service in test environment)
    payment_result = payment_client.process_payment(
        order_id=order.id,
        amount=order.total,
        payment_method='credit_card'
    )
    
    # Verify payment was recorded
    assert payment_result.status == 'completed'
    
    # Verify order was updated
    updated_order = get_order(order.id)
    assert updated_order.payment_status == 'paid'
```

**4. Automate contract publishing in CI**:

```yaml
# .github/workflows/contracts.yml
name: Publish Contracts

on:
  push:
    branches: [main]

jobs:
  publish-contracts:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Run contract tests
        run: pytest tests/contracts/ -v
      - name: Publish contracts to broker
        run: |
          pact-broker publish pacts/ \
            --consumer-app-version=${{ github.sha }} \
            --branch=${{ github.ref_name }} \
            --broker-base-url=${{ secrets.PACT_BROKER_URL }} \
            --broker-token=${{ secrets.PACT_BROKER_TOKEN }}
```

**5. Document your contracts**:

Pact Broker provides a web UI showing all contracts and their verification status. Use it as living documentation for your microservices architecture.

## Property-Based Testing with Hypothesis

## The Example-Based Testing Limitation

Traditional tests use specific examples:

```python
# tests/test_string_utils.py
def test_reverse_string():
    assert reverse("hello") == "olleh"
    assert reverse("world") == "dlrow"
    assert reverse("") == ""
```

This works, but what about:
- Unicode characters?
- Very long strings?
- Strings with special characters?
- Edge cases you didn't think of?

You can't test every possible input. But you can test **properties** that should hold for all inputs.

### Iteration 1: Introduction to Property-Based Testing

Instead of testing specific examples, test properties:

```python
# tests/test_string_utils_property.py
from hypothesis import given
from hypothesis import strategies as st

@given(st.text())
def test_reverse_string_property(s):
    """
    Property: Reversing a string twice returns the original string.
    
    This should be true for ANY string.
    """
    assert reverse(reverse(s)) == s
```

Run the property test:

```bash
$ pytest tests/test_string_utils_property.py -v
```

**Output**:

```text
tests/test_string_utils_property.py::test_reverse_string_property PASSED

Hypothesis ran 100 examples
```

Hypothesis automatically generated 100 different strings and verified the property holds for all of them.

**What if the property doesn't hold?** Let's test a buggy implementation:

```python
# src/string_utils.py (buggy version)
def reverse(s):
    """Reverse a string (buggy implementation)."""
    if len(s) > 1000:
        return s  # Bug: Don't reverse very long strings
    return s[::-1]
```

Run the property test:

```bash
$ pytest tests/test_string_utils_property.py -v
```

**Output**:

```text
tests/test_string_utils_property.py::test_reverse_string_property FAILED

E   AssertionError: assert 'aaa...aaa' == 'aaa...aaa'
E   
E   Falsifying example: test_reverse_string_property(
E       s='a' * 1001
E   )
E   
E   You can reproduce this example by temporarily adding @reproduce_failure('6.92.1', b'...')
E   as a decorator on your test case
```

**Hypothesis found the bug!** It automatically generated a string longer than 1000 characters and discovered the property doesn't hold.

### Iteration 2: Testing Complex Data Structures

Let's test our API client with property-based testing:

```python
# tests/test_api_client_properties.py
from hypothesis import given, assume
from hypothesis import strategies as st
from src.api_client import APIClient
from unittest.mock import Mock, patch

# Define strategies for generating test data
user_strategy = st.fixed_dictionaries({
    'id': st.integers(min_value=1, max_value=1000000),
    'username': st.text(
        alphabet=st.characters(whitelist_categories=('Lu', 'Ll', 'Nd')),
        min_size=3,
        max_size=20
    ),
    'email': st.emails()
})

@given(user_strategy)
def test_get_user_returns_valid_user(user_data):
    """
    Property: get_user should always return a user with valid structure.
    """
    with patch('src.api_client.requests.Session') as mock_session_class:
        mock_session = Mock()
        mock_session_class.return_value = mock_session
        mock_session.get.return_value.json.return_value = user_data
        
        client = APIClient("https://api.example.com", "test-key")
        result = client.get_user(user_data['id'])
        
        # Properties that should always hold
        assert 'id' in result
        assert 'username' in result
        assert 'email' in result
        assert isinstance(result['id'], int)
        assert isinstance(result['username'], str)
        assert '@' in result['email']
```

Run the property test:

```bash
$ pytest tests/test_api_client_properties.py -v
```

**Output**:

```text
tests/test_api_client_properties.py::test_get_user_returns_valid_user PASSED

Hypothesis ran 100 examples
```

### Iteration 3: Finding Edge Cases Automatically

Property-based testing excels at finding edge cases. Let's test order processing:

```python
# src/order_calculator.py
from decimal import Decimal

def calculate_order_total(items, tax_rate=Decimal('0.0725')):
    """Calculate order total with tax."""
    subtotal = sum(
        Decimal(str(item['price'])) * item['quantity']
        for item in items
    )
    tax = subtotal * tax_rate
    return subtotal + tax
```

Test with properties:

```python
# tests/test_order_calculator_properties.py
from hypothesis import given, assume
from hypothesis import strategies as st
from decimal import Decimal
from src.order_calculator import calculate_order_total

item_strategy = st.fixed_dictionaries({
    'price': st.floats(min_value=0.01, max_value=10000.0),
    'quantity': st.integers(min_value=1, max_value=100)
})

@given(st.lists(item_strategy, min_size=1, max_size=10))
def test_order_total_is_positive(items):
    """Property: Order total should always be positive."""
    total = calculate_order_total(items)
    assert total > 0

@given(st.lists(item_strategy, min_size=1, max_size=10))
def test_order_total_increases_with_quantity(items):
    """Property: Increasing quantity should increase total."""
    original_total = calculate_order_total(items)
    
    # Double the quantity of the first item
    items_doubled = items.copy()
    items_doubled[0] = {
        'price': items[0]['price'],
        'quantity': items[0]['quantity'] * 2
    }
    
    doubled_total = calculate_order_total(items_doubled)
    assert doubled_total > original_total

@given(st.lists(item_strategy, min_size=1, max_size=10))
def test_order_total_with_zero_tax_equals_subtotal(items):
    """Property: With 0% tax, total should equal subtotal."""
    total = calculate_order_total(items, tax_rate=Decimal('0'))
    
    expected_subtotal = sum(
        Decimal(str(item['price'])) * item['quantity']
        for item in items
    )
    
    assert abs(total - expected_subtotal) < Decimal('0.01')
```

Run the property tests:

```bash
$ pytest tests/test_order_calculator_properties.py -v
```

**Output**:

```text
tests/test_order_calculator_properties.py::test_order_total_is_positive FAILED

E   AssertionError: assert Decimal('0') > 0
E   
E   Falsifying example: test_order_total_is_positive(
E       items=[{'price': 0.01, 'quantity': 1}]
E   )
```

**Bug found!** With very small prices and rounding, the total can become zero. This is an edge case we wouldn't have thought to test manually.

### Iteration 4: Stateful Testing

Test sequences of operations:

```python
# tests/test_shopping_cart_stateful.py
from hypothesis.stateful import RuleBasedStateMachine, rule, invariant
from hypothesis import strategies as st
from src.shopping_cart import ShoppingCart

class ShoppingCartStateMachine(RuleBasedStateMachine):
    """
    Stateful testing: Generate random sequences of cart operations
    and verify invariants always hold.
    """
    
    def __init__(self):
        super().__init__()
        self.cart = ShoppingCart()
        self.expected_items = {}
    
    @rule(product_id=st.integers(min_value=1, max_value=100),
          quantity=st.integers(min_value=1, max_value=10))
    def add_item(self, product_id, quantity):
        """Add an item to the cart."""
        self.cart.add_item(product_id, quantity)
        self.expected_items[product_id] = self.expected_items.get(product_id, 0) + quantity
    
    @rule(product_id=st.integers(min_value=1, max_value=100))
    def remove_item(self, product_id):
        """Remove an item from the cart."""
        if product_id in self.expected_items:
            self.cart.remove_item(product_id)
            del self.expected_items[product_id]
    
    @rule(product_id=st.integers(min_value=1, max_value=100),
          quantity=st.integers(min_value=1, max_value=10))
    def update_quantity(self, product_id, quantity):
        """Update item quantity."""
        if product_id in self.expected_items:
            self.cart.update_quantity(product_id, quantity)
            self.expected_items[product_id] = quantity
    
    @rule()
    def clear_cart(self):
        """Clear the cart."""
        self.cart.clear()
        self.expected_items.clear()
    
    @invariant()
    def cart_matches_expected(self):
        """Invariant: Cart contents should always match expected state."""
        actual_items = self.cart.get_items()
        assert actual_items == self.expected_items
    
    @invariant()
    def cart_total_is_consistent(self):
        """Invariant: Cart total should equal sum of item totals."""
        expected_total = sum(
            self.cart.get_item_price(product_id) * quantity
            for product_id, quantity in self.expected_items.items()
        )
        assert abs(self.cart.get_total() - expected_total) < 0.01

# Run the state machine
TestShoppingCart = ShoppingCartStateMachine.TestCase
```

Run the stateful test:

```bash
$ pytest tests/test_shopping_cart_stateful.py -v
```

**Output**:

```text
tests/test_shopping_cart_stateful.py::TestShoppingCart::test_shopping_cart_state_machine FAILED

E   AssertionError: Cart total doesn't match expected
E   
E   Falsifying example:
E       state = ShoppingCartStateMachine()
E       state.add_item(product_id=1, quantity=5)
E       state.update_quantity(product_id=1, quantity=3)
E       state.add_item(product_id=1, quantity=2)
E       # Cart total: 10.00, Expected: 15.00
E   
E   Minimal failing sequence found after 47 steps
```

**Bug found!** When updating quantity and then adding more items, the cart total calculation is incorrect. Hypothesis automatically found the minimal sequence of operations that triggers the bug.

### When to Use Property-Based Testing

**Optimal scenarios**:
- Testing pure functions (no side effects)
- Testing data transformations
- Testing parsers and serializers
- Testing mathematical properties
- Finding edge cases in complex logic

**When to avoid**:
- Testing UI interactions (too stateful)
- Testing external integrations (non-deterministic)
- Testing business rules that don't have clear properties
- When example-based tests are clearer and sufficient

### Pro Tips for Property-Based Testing

**1. Start with simple properties**:

Don't try to test everything with properties. Start with obvious invariants:
- Reversing twice returns original
- Sorting is idempotent (sorting twice = sorting once)
- Encoding then decoding returns original
- Adding then removing returns original state

**2. Combine with example-based tests**:

```python
# tests/test_string_utils_combined.py
import pytest
from hypothesis import given
from hypothesis import strategies as st

# Example-based tests for specific cases
def test_reverse_empty_string():
    assert reverse("") == ""

def test_reverse_single_char():
    assert reverse("a") == "a"

def test_reverse_palindrome():
    assert reverse("racecar") == "racecar"

# Property-based test for general behavior
@given(st.text())
def test_reverse_property(s):
    assert reverse(reverse(s)) == s
```

**3. Use `assume()` to filter invalid inputs**:

```python
from hypothesis import given, assume
from hypothesis import strategies as st

@given(st.integers(), st.integers())
def test_division_property(a, b):
    """Property: (a / b) * b should equal a."""
    assume(b != 0)  # Filter out division by zero
    
    result = a / b
    assert abs(result * b - a) < 0.0001
```

**4. Shrink failing examples**:

Hypothesis automatically finds the minimal failing example:

```python
@given(st.lists(st.integers()))
def test_list_property(lst):
    """Property: Sorting should not change list length."""
    sorted_lst = sorted(lst)
    assert len(sorted_lst) == len(lst)
```

If this fails, Hypothesis will shrink the failing example from a large list to the smallest list that still fails.

**5. Use `@reproduce_failure` to debug**:

When Hypothesis finds a failure, it provides a decorator to reproduce it:

```python
from hypothesis import reproduce_failure

@reproduce_failure('6.92.1', b'AXicY2BgYGAAAQAA')
@given(st.text())
def test_reverse_property(s):
    assert reverse(reverse(s)) == s
```

This runs the exact same failing example every time, making debugging easier.

**6. Configure Hypothesis for different scenarios**:

```python
# conftest.py
from hypothesis import settings, Verbosity

# Default settings for all tests
settings.register_profile("default", max_examples=100)

# Thorough testing for CI
settings.register_profile("ci", max_examples=1000, deadline=None)

# Quick testing for development
settings.register_profile("dev", max_examples=10)

# Debug mode
settings.register_profile("debug", max_examples=10, verbosity=Verbosity.verbose)

# Activate profile based on environment
import os
settings.load_profile(os.getenv("HYPOTHESIS_PROFILE", "default"))
```

Use different profiles:

```bash
# Quick feedback during development
$ HYPOTHESIS_PROFILE=dev pytest

# Thorough testing in CI
$ HYPOTHESIS_PROFILE=ci pytest

# Debug a failing test
$ HYPOTHESIS_PROFILE=debug pytest tests/test_specific.py -v
```

## Best Practices Summary

After 19 chapters of pytest techniques, let's distill the essential principles that separate good tests from great tests. These aren't arbitrary rules—they're battle-tested patterns that make tests maintainable, reliable, and valuable.

## The Complexity Trap

Tests are code. Code can become complex. But complex tests defeat their purpose: they're hard to understand, hard to maintain, and hard to trust.

**Bad example** (complex test):

```python
# tests/test_order_processing_complex.py
def test_order_processing():
    """Test order processing (too complex)."""
    # Setup (50 lines of configuration)
    db = setup_database()
    cache = setup_cache()
    email_service = setup_email_service()
    payment_gateway = setup_payment_gateway()
    inventory_system = setup_inventory_system()
    
    # Create test data (30 lines)
    user = create_user(db, username="testuser", email="test@example.com")
    products = [
        create_product(db, name=f"Product {i}", price=10.0 * i)
        for i in range(1, 11)
    ]
    
    # Test multiple scenarios in one test (100 lines)
    for product in products:
        for quantity in [1, 5, 10]:
            for payment_method in ['credit_card', 'paypal', 'bank_transfer']:
                for shipping_method in ['standard', 'express', 'overnight']:
                    order = create_order(
                        user=user,
                        items=[{'product': product, 'quantity': quantity}],
                        payment_method=payment_method,
                        shipping_method=shipping_method
                    )
                    
                    result = process_order(
                        order,
                        db=db,
                        cache=cache,
                        email_service=email_service,
                        payment_gateway=payment_gateway,
                        inventory_system=inventory_system
                    )
                    
                    # Assertions (20 lines of complex validation)
                    assert result.status == 'completed'
                    assert result.payment_status == 'paid'
                    assert result.shipping_status == 'pending'
                    # ... 17 more assertions
```

**Problems**:
- 200+ lines in one test
- Tests multiple scenarios simultaneously
- When it fails, you don't know which scenario broke
- Hard to understand what's being tested
- Difficult to modify without breaking something

**Good example** (simple, focused tests):

```python
# tests/test_order_processing_simple.py
import pytest
from src.order_processor import process_order

@pytest.fixture
def basic_order(db_session):
    """Create a basic order for testing."""
    user = create_user(db_session, username="testuser")
    product = create_product(db_session, name="Widget", price=10.0)
    return create_order(
        user=user,
        items=[{'product': product, 'quantity': 1}]
    )

def test_process_order_marks_as_completed(basic_order, order_processor):
    """Test that processing an order marks it as completed."""
    result = order_processor.process(basic_order)
    
    assert result.status == 'completed'

def test_process_order_charges_payment(basic_order, order_processor, mock_payment_gateway):
    """Test that processing an order charges the payment method."""
    order_processor.process(basic_order)
    
    mock_payment_gateway.charge.assert_called_once_with(
        amount=10.0,
        payment_method=basic_order.payment_method
    )

def test_process_order_sends_confirmation_email(basic_order, order_processor, mock_email_service):
    """Test that processing an order sends a confirmation email."""
    order_processor.process(basic_order)
    
    mock_email_service.send.assert_called_once()
    email_call = mock_email_service.send.call_args
    assert 'Order Confirmation' in email_call[1]['subject']

def test_process_order_updates_inventory(basic_order, order_processor, db_session):
    """Test that processing an order reduces inventory."""
    product = basic_order.items[0].product
    original_quantity = product.inventory_quantity
    
    order_processor.process(basic_order)
    
    db_session.refresh(product)
    assert product.inventory_quantity == original_quantity - 1
```

**Improvements**:
- Each test is 5-10 lines
- Each test has a single, clear purpose
- Test names describe exactly what's being tested
- When a test fails, you immediately know what broke
- Easy to add new tests without affecting existing ones

### Principles for Simple Tests

**1. One concept per test**:

Test one behavior, one edge case, one error condition. Not all of them at once.

**2. Arrange-Act-Assert structure**:

```python
def test_example():
    # Arrange: Set up test data
    user = create_user(username="alice")
    
    # Act: Perform the action being tested
    result = authenticate(user, password="correct_password")
    
    # Assert: Verify the outcome
    assert result.is_authenticated
```

**3. Minimize setup code**:

Use fixtures to hide complex setup:

```python
# Bad: Setup repeated in every test
def test_user_login():
    db = setup_database()
    user = create_user(db, username="alice", email="alice@example.com")
    session = create_session(db)
    # ... test code

# Good: Setup in fixture
@pytest.fixture
def authenticated_user(db_session):
    return create_user(db_session, username="alice")

def test_user_login(authenticated_user):
    # Test code starts immediately
    result = authenticate(authenticated_user, password="correct")
    assert result.is_authenticated
```

**4. Avoid test helpers that obscure behavior**:

```python
# Bad: Helper hides what's being tested
def test_order_processing():
    order = create_test_order_with_all_options()  # What options?
    result = process_and_verify_order(order)      # What's being verified?
    assert result  # What does True mean?

# Good: Explicit test
def test_order_processing():
    order = create_order(
        items=[{'product_id': 1, 'quantity': 2}],
        payment_method='credit_card'
    )
    result = process_order(order)
    assert result.status == 'completed'
    assert result.payment_status == 'paid'
```

**5. Use descriptive variable names**:

```python
# Bad: Cryptic names
def test_x():
    a = f(b, c)
    assert a == d

# Good: Clear names
def test_calculate_order_total_with_tax():
    order_subtotal = Decimal('100.00')
    tax_rate = Decimal('0.0725')
    
    total = calculate_total(order_subtotal, tax_rate)
    
    expected_total = Decimal('107.25')
    assert total == expected_total
```

## One Assertion Per Test (Usually)

## The Multiple Assertion Debate

Should tests have one assertion or multiple? The answer: **it depends on what you're testing**.

**The principle**: One **logical assertion** per test. This might be one `assert` statement, or it might be several `assert` statements that together verify one concept.

**Good: Multiple assertions for one concept**:

```python
def test_create_user_returns_valid_user_object():
    """Test that create_user returns a properly structured user."""
    user = create_user(username="alice", email="alice@example.com")
    
    # These assertions all verify the same concept:
    # "The returned object is a valid user"
    assert user.username == "alice"
    assert user.email == "alice@example.com"
    assert user.id is not None
    assert user.created_at is not None
    assert isinstance(user.created_at, datetime)
```

**Bad: Multiple unrelated assertions**:

```python
def test_user_operations():
    """Test various user operations (too broad)."""
    # Creating a user
    user = create_user(username="alice")
    assert user.username == "alice"
    
    # Updating a user
    update_user(user, email="newemail@example.com")
    assert user.email == "newemail@example.com"
    
    # Deleting a user
    delete_user(user)
    assert get_user(user.id) is None
    
    # User authentication
    auth_result = authenticate("alice", "password")
    assert auth_result.is_authenticated
```

**Problem**: If the second assertion fails, you never run the third and fourth. You don't know if deletion and authentication work.

**Better: Split into focused tests**:

```python
def test_create_user_sets_username():
    user = create_user(username="alice")
    assert user.username == "alice"

def test_update_user_changes_email():
    user = create_user(username="alice")
    update_user(user, email="newemail@example.com")
    assert user.email == "newemail@example.com"

def test_delete_user_removes_from_database():
    user = create_user(username="alice")
    delete_user(user)
    assert get_user(user.id) is None

def test_authenticate_user_with_valid_credentials():
    create_user(username="alice", password="password")
    result = authenticate("alice", "password")
    assert result.is_authenticated
```

### When Multiple Assertions Are Appropriate

**1. Verifying object structure**:

```python
def test_api_response_has_required_fields():
    """Test that API response contains all required fields."""
    response = api_client.get_user(1)
    
    # All assertions verify the same concept: "response structure is valid"
    assert 'id' in response
    assert 'username' in response
    assert 'email' in response
    assert 'created_at' in response
```

**2. Verifying state transitions**:

```python
def test_order_processing_updates_all_related_entities():
    """Test that processing an order updates order, inventory, and payment."""
    order = create_order(items=[{'product_id': 1, 'quantity': 2}])
    product = get_product(1)
    original_inventory = product.inventory_quantity
    
    process_order(order)
    
    # All assertions verify the same concept: "order processing updates everything"
    assert order.status == 'completed'
    assert product.inventory_quantity == original_inventory - 2
    assert order.payment_status == 'paid'
```

**3. Verifying invariants**:

```python
def test_shopping_cart_maintains_invariants():
    """Test that cart operations maintain consistency."""
    cart = ShoppingCart()
    cart.add_item(product_id=1, quantity=2, price=10.0)
    cart.add_item(product_id=2, quantity=1, price=20.0)
    
    # All assertions verify the same concept: "cart is internally consistent"
    assert cart.item_count == 3
    assert cart.subtotal == 40.0
    assert len(cart.items) == 2
```

### When to Split Assertions

**Split when assertions test different behaviors**:

```python
# Bad: Testing two different behaviors
def test_user_creation_and_authentication():
    user = create_user(username="alice", password="password")
    assert user.id is not None  # Tests creation
    
    result = authenticate("alice", "password")
    assert result.is_authenticated  # Tests authentication

# Good: Separate tests for separate behaviors
def test_create_user_assigns_id():
    user = create_user(username="alice", password="password")
    assert user.id is not None

def test_authenticate_user_with_valid_credentials():
    create_user(username="alice", password="password")
    result = authenticate("alice", "password")
    assert result.is_authenticated
```

### Using pytest-subtests for Related Assertions

When you have many related assertions, use subtests to run all of them even if some fail:

```python
def test_api_response_fields(subtests):
    """Test that API response has all required fields."""
    response = api_client.get_user(1)
    
    required_fields = ['id', 'username', 'email', 'created_at', 'updated_at']
    
    for field in required_fields:
        with subtests.test(field=field):
            assert field in response, f"Missing required field: {field}"
```

**Output if multiple fields are missing**:

```text
test_api_response_fields SUBFAIL [email]
test_api_response_fields SUBFAIL [updated_at]

E   AssertionError: Missing required field: email
E   AssertionError: Missing required field: updated_at
```

You see all failures at once, not just the first one.

## Name Tests to Describe What They Test

## The Naming Problem

Test names are documentation. When a test fails in CI, the name is often the first (and sometimes only) thing you see. A good name tells you exactly what broke.

**Bad names**:

```python
def test_user():
    pass

def test_user_2():
    pass

def test_order():
    pass

def test_1():
    pass

def test_edge_case():
    pass
```

**When these fail, you have no idea what broke.**

**Good names**:

```python
def test_create_user_with_valid_email_succeeds():
    pass

def test_create_user_with_invalid_email_raises_validation_error():
    pass

def test_process_order_with_insufficient_inventory_raises_out_of_stock_error():
    pass

def test_calculate_order_total_includes_tax():
    pass

def test_authenticate_user_with_expired_token_returns_unauthorized():
    pass
```

**When these fail, you know exactly what to investigate.**

### Naming Patterns

**Pattern 1: test_[function]_[scenario]_[expected_result]**:

```python
def test_authenticate_user_with_valid_credentials_returns_token():
    pass

def test_authenticate_user_with_invalid_password_raises_authentication_error():
    pass

def test_authenticate_user_with_expired_token_returns_unauthorized():
    pass
```

**Pattern 2: test_[action]_[condition]**:

```python
def test_payment_processing_succeeds_with_valid_card():
    pass

def test_payment_processing_fails_with_insufficient_funds():
    pass

def test_payment_processing_retries_on_network_error():
    pass
```

**Pattern 3: test_[expected_behavior]_when_[condition]**:

```python
def test_order_total_includes_tax_when_shipping_to_california():
    pass

def test_order_total_excludes_tax_when_shipping_internationally():
    pass

def test_discount_applies_when_order_exceeds_minimum():
    pass
```

### Real-World Example

**Bad naming** (actual test suite I encountered):

```python
def test_1():
    """Test user creation."""
    pass

def test_2():
    """Test user update."""
    pass

def test_3():
    """Test user deletion."""
    pass

def test_4():
    """Test edge case."""
    pass
```

**CI output when test_3 fails**:

```text
FAILED tests/test_users.py::test_3
```

You have to open the file and read the docstring to understand what failed.

**Good naming** (refactored):

```python
def test_create_user_with_valid_data_succeeds():
    pass

def test_update_user_email_changes_email_address():
    pass

def test_delete_user_removes_user_from_database():
    pass

def test_delete_user_with_active_orders_raises_integrity_error():
    pass
```

**CI output when test fails**:

```text
FAILED tests/test_users.py::test_delete_user_with_active_orders_raises_integrity_error
```

You immediately know: "Deleting a user with active orders should raise an integrity error, but it didn't."

### Naming for Parametrized Tests

Use `ids` parameter to make parametrized tests readable:

```python
# Bad: Cryptic test IDs
@pytest.mark.parametrize("status_code", [400, 401, 403, 404, 500])
def test_http_errors(status_code):
    pass
```

**Output**:

```text
test_http_errors[400] PASSED
test_http_errors[401] PASSED
test_http_errors[403] FAILED
```

What does 403 mean in this context?

**Good: Descriptive test IDs**:

```python
@pytest.mark.parametrize("status_code,error_name", [
    (400, "bad_request"),
    (401, "unauthorized"),
    (403, "forbidden"),
    (404, "not_found"),
    (500, "internal_server_error"),
], ids=lambda x: x[1] if isinstance(x, tuple) else str(x))
def test_http_errors(status_code, error_name):
    pass
```

**Output**:

```text
test_http_errors[bad_request] PASSED
test_http_errors[unauthorized] PASSED
test_http_errors[forbidden] FAILED
```

Now you know exactly which error case failed.

### Pro Tips for Test Naming

**1. Use domain language, not implementation details**:

```python
# Bad: Implementation-focused
def test_database_query_returns_list():
    pass

# Good: Behavior-focused
def test_search_users_returns_matching_users():
    pass
```

**2. Include the failure condition**:

```python
# Bad: Doesn't say what should happen
def test_invalid_email():
    pass

# Good: Says what should happen
def test_create_user_with_invalid_email_raises_validation_error():
    pass
```

**3. Be specific about edge cases**:

```python
# Bad: Vague
def test_edge_case():
    pass

# Good: Specific
def test_calculate_discount_with_zero_quantity_returns_zero():
    pass
```

**4. Use consistent naming across the test suite**:

Pick a pattern and stick to it. Don't mix:
- `test_user_creation_succeeds`
- `test_when_updating_user_email_changes`
- `test_delete_user`

Choose one pattern for the entire project.

## Avoid Test Interdependency

## The Interdependency Problem

Tests should be independent. Each test should run successfully in isolation, in any order, and regardless of which other tests run.

**Bad: Tests depend on execution order**:

```python
# tests/test_user_lifecycle.py (BAD)
_created_user_id = None

def test_1_create_user():
    """Create a user (must run first)."""
    global _created_user_id
    user = create_user(username="alice")
    _created_user_id = user.id
    assert user.id is not None

def test_2_update_user():
    """Update the user (must run after test_1)."""
    global _created_user_id
    assert _created_user_id is not None, "test_1_create_user must run first"
    
    update_user(_created_user_id, email="newemail@example.com")
    user = get_user(_created_user_id)
    assert user.email == "newemail@example.com"

def test_3_delete_user():
    """Delete the user (must run after test_2)."""
    global _created_user_id
    assert _created_user_id is not None, "test_1_create_user must run first"
    
    delete_user(_created_user_id)
    assert get_user(_created_user_id) is None
```

**Problems**:
- Tests must run in order (1, 2, 3)
- If test_1 fails, test_2 and test_3 also fail
- Can't run test_2 or test_3 in isolation
- Parallel execution breaks everything
- Debugging is difficult

**Run tests in random order** (they fail):

```bash
$ pytest tests/test_user_lifecycle.py --random-order
```

**Output**:

```text
test_user_lifecycle.py::test_3_delete_user FAILED
test_user_lifecycle.py::test_1_create_user PASSED
test_user_lifecycle.py::test_2_update_user FAILED

E   AssertionError: test_1_create_user must run first
```

**Good: Independent tests**:

```python
# tests/test_user_lifecycle.py (GOOD)
@pytest.fixture
def created_user(db_session):
    """Create a user for testing."""
    user = create_user(username="alice")
    yield user
    # Cleanup happens automatically after test

def test_create_user_assigns_id(db_session):
    """Test that creating a user assigns an ID."""
    user = create_user(username="alice")
    assert user.id is not None

def test_update_user_changes_email(created_user, db_session):
    """Test that updating a user changes their email."""
    update_user(created_user.id, email="newemail@example.com")
    
    updated_user = get_user(created_user.id)
    assert updated_user.email == "newemail@example.com"

def test_delete_user_removes_from_database(created_user, db_session):
    """Test that deleting a user removes them from the database."""
    delete_user(created_user.id)
    assert get_user(created_user.id) is None
```

**Improvements**:
- Each test is self-contained
- Tests can run in any order
- Tests can run in parallel
- Each test can be run in isolation
- If one test fails, others still run

**Run tests in random order** (they pass):

```bash
$ pytest tests/test_user_lifecycle.py --random-order
```

**Output**:

```text
test_user_lifecycle.py::test_delete_user_removes_from_database PASSED
test_user_lifecycle.py::test_create_user_assigns_id PASSED
test_user_lifecycle.py::test_update_user_changes_email PASSED

======================== 3 passed in 0.23s ========================
```

### Common Sources of Test Interdependency

**1. Shared global state**:

```python
# Bad: Global state
_cache = {}

def test_cache_stores_value():
    _cache['key'] = 'value'
    assert _cache['key'] == 'value'

def test_cache_is_empty():
    # Fails if test_cache_stores_value ran first
    assert len(_cache) == 0
```

**Fix: Use fixtures to isolate state**:

```python
@pytest.fixture
def cache():
    """Provide a fresh cache for each test."""
    return {}

def test_cache_stores_value(cache):
    cache['key'] = 'value'
    assert cache['key'] == 'value'

def test_cache_is_empty(cache):
    assert len(cache) == 0
```

**2. Database state**:

```python
# Bad: Tests modify shared database
def test_create_user():
    create_user(username="alice")
    assert get_user_by_username("alice") is not None

def test_user_does_not_exist():
    # Fails if test_create_user ran first
    assert get_user_by_username("alice") is None
```

**Fix: Use transactions or database cleanup**:

```python
@pytest.fixture
def db_session():
    """Provide a database session that rolls back after each test."""
    session = create_session()
    yield session
    session.rollback()
    session.close()

def test_create_user(db_session):
    create_user(db_session, username="alice")
    assert get_user_by_username(db_session, "alice") is not None

def test_user_does_not_exist(db_session):
    assert get_user_by_username(db_session, "alice") is None
```

**3. File system state**:

```python
# Bad: Tests modify shared files
def test_write_config():
    write_config_file("config.json", {"key": "value"})
    assert os.path.exists("config.json")

def test_config_file_does_not_exist():
    # Fails if test_write_config ran first
    assert not os.path.exists("config.json")
```

**Fix: Use temporary directories**:

```python
def test_write_config(tmp_path):
    config_file = tmp_path / "config.json"
    write_config_file(config_file, {"key": "value"})
    assert config_file.exists()

def test_config_file_does_not_exist(tmp_path):
    config_file = tmp_path / "config.json"
    assert not config_file.exists()
```

**4. Time-dependent tests**:

```python
# Bad: Tests depend on current time
def test_token_expires_after_one_hour():
    token = create_token(expires_in=3600)
    time.sleep(3601)  # Wait for expiration
    assert is_token_expired(token)
```

**Fix: Mock time or use explicit timestamps**:

```python
def test_token_expires_after_one_hour(freezer):
    """Test token expiration (using pytest-freezegun)."""
    token = create_token(expires_in=3600)
    
    # Fast-forward time
    freezer.move_to(datetime.now() + timedelta(hours=1, seconds=1))
    
    assert is_token_expired(token)
```

### Detecting Test Interdependency

**Run tests in random order**:

```bash
$ pip install pytest-random-order
$ pytest --random-order
```

If tests fail with random order but pass with normal order, you have interdependency.

**Run tests in isolation**:

```bash
# Run each test individually
$ pytest tests/test_users.py::test_create_user
$ pytest tests/test_users.py::test_update_user
$ pytest tests/test_users.py::test_delete_user
```

If a test passes when run with others but fails in isolation, it depends on another test.

**Run tests in parallel**:

```bash
$ pytest -n auto
```

If tests fail with parallel execution but pass sequentially, they share state.

## Use Fixtures for Setup, Not Test Data

## The Fixture Misuse Problem

Fixtures are for **setup and teardown**, not for defining test data. Test data should be visible in the test itself.

**Bad: Test data hidden in fixtures**:

```python
# conftest.py (BAD)
@pytest.fixture
def user():
    """Create a user with specific attributes."""
    return create_user(
        username="alice",
        email="alice@example.com",
        age=30,
        country="US",
        subscription_type="premium",
        account_balance=100.00
    )

# tests/test_user.py
def test_user_can_purchase_premium_content(user):
    """Test that user can purchase premium content."""
    result = purchase_content(user, content_id=123)
    assert result.success
```

**Problem**: When this test fails, you have to look at the fixture to understand why. What attributes of the user matter for this test? Is it the subscription type? The account balance? The country?

**Good: Test data visible in test**:

```python
# conftest.py (GOOD)
@pytest.fixture
def db_session():
    """Provide a database session (setup only)."""
    session = create_session()
    yield session
    session.rollback()
    session.close()

# tests/test_user.py
def test_premium_user_can_purchase_premium_content(db_session):
    """Test that premium users can purchase premium content."""
    # Test data is visible here
    user = create_user(
        db_session,
        username="alice",
        subscription_type="premium",
        account_balance=100.00
    )
    
    result = purchase_content(user, content_id=123)
    
    assert result.success

def test_free_user_cannot_purchase_premium_content(db_session):
    """Test that free users cannot purchase premium content."""
    # Different test data, clearly visible
    user = create_user(
        db_session,
        username="bob",
        subscription_type="free",
        account_balance=100.00
    )
    
    result = purchase_content(user, content_id=123)
    
    assert not result.success
    assert result.error == "Premium subscription required"
```

**Improvements**:
- Test data is visible in the test
- Each test creates exactly the data it needs
- When a test fails, you can see immediately what data was used
- Tests are self-documenting

### When to Use Fixtures for Data

**Use fixtures for data when**:

1. **The data is complex to create**:

```python
@pytest.fixture
def complex_order(db_session):
    """Create an order with multiple items, discounts, and shipping."""
    user = create_user(db_session)
    products = [create_product(db_session) for _ in range(5)]
    order = create_order(db_session, user=user)
    
    for product in products:
        add_order_item(order, product, quantity=2)
    
    apply_discount(order, code="SAVE10")
    set_shipping_address(order, address=create_address())
    
    return order
```

2. **The data is reused across many tests**:

```python
@pytest.fixture
def sample_products(db_session):
    """Create a standard set of products for testing."""
    return [
        create_product(db_session, name="Widget", price=10.0),
        create_product(db_session, name="Gadget", price=20.0),
        create_product(db_session, name="Doohickey", price=30.0),
    ]
```

3. **The data represents a standard scenario**:

```python
@pytest.fixture
def authenticated_user(db_session):
    """Create a user and authenticate them."""
    user = create_user(db_session, username="alice")
    token = authenticate(user, password="password")
    return user, token
```

### Factory Fixtures: The Best of Both Worlds

Use factory fixtures to combine fixture convenience with test-visible data:

```python
# conftest.py
@pytest.fixture
def user_factory(db_session):
    """Factory for creating users with custom attributes."""
    def _create_user(**kwargs):
        defaults = {
            'username': 'testuser',
            'email': 'test@example.com',
            'subscription_type': 'free',
            'account_balance': 0.00
        }
        defaults.update(kwargs)
        return create_user(db_session, **defaults)
    
    return _create_user

# tests/test_user.py
def test_premium_user_can_purchase_premium_content(user_factory):
    """Test that premium users can purchase premium content."""
    # Create user with specific attributes (visible in test)
    user = user_factory(
        subscription_type="premium",
        account_balance=100.00
    )
    
    result = purchase_content(user, content_id=123)
    assert result.success

def test_user_with_insufficient_balance_cannot_purchase(user_factory):
    """Test that users with insufficient balance cannot purchase."""
    # Create user with different attributes (visible in test)
    user = user_factory(
        subscription_type="premium",
        account_balance=5.00  # Not enough for $10 content
    )
    
    result = purchase_content(user, content_id=123)
    assert not result.success
    assert result.error == "Insufficient balance"
```

**Benefits**:
- Fixture handles the complex creation logic
- Test data is still visible in the test
- Easy to create variations for different test scenarios
- Reduces duplication without hiding information

### Pro Tips for Fixture Usage

**1. Name fixtures by what they provide, not what they do**:

```python
# Bad: Named by action
@pytest.fixture
def setup_database():
    pass

# Good: Named by what it provides
@pytest.fixture
def db_session():
    pass
```

**2. Use fixture scopes appropriately**:

```python
# Function scope (default): New instance for each test
@pytest.fixture
def user(db_session):
    return create_user(db_session)

# Module scope: Shared across all tests in the module
@pytest.fixture(scope="module")
def database_connection():
    conn = create_connection()
    yield conn
    conn.close()

# Session scope: Shared across entire test session
@pytest.fixture(scope="session")
def test_config():
    return load_config("test_config.yaml")
```

**3. Use autouse sparingly**:

```python
# Bad: Autouse for everything
@pytest.fixture(autouse=True)
def setup_everything():
    # Runs before every test, even if not needed
    pass

# Good: Autouse only for essential setup
@pytest.fixture(autouse=True)
def reset_global_state():
    """Reset global state before each test."""
    global_cache.clear()
    yield
    global_cache.clear()
```

**4. Document fixture purpose and usage**:

```python
@pytest.fixture
def authenticated_api_client(db_session):
    """
    Provide an API client authenticated as a test user.
    
    Creates a test user, authenticates them, and returns a client
    with the authentication token set.
    
    Usage:
        def test_api_endpoint(authenticated_api_client):
            response = authenticated_api_client.get('/api/users/me')
            assert response.status_code == 200
    """
    user = create_user(db_session, username="testuser")
    token = authenticate(user, password="password")
    client = APIClient(token=token)
    return client
```

## Common Pitfalls to Avoid

These are the mistakes that even experienced developers make. Learn to recognize and avoid them.

## The Mocking Trap

Mocks are powerful, but overuse leads to tests that pass even when the code is broken.

**Bad: Over-mocked test**:

```python
# tests/test_order_processor_overmocked.py
from unittest.mock import Mock, patch

def test_process_order():
    """Test order processing (over-mocked)."""
    # Mock everything
    mock_db = Mock()
    mock_payment = Mock()
    mock_email = Mock()
    mock_inventory = Mock()
    mock_analytics = Mock()
    
    # Configure mocks
    mock_db.get_user.return_value = Mock(id=1, email="test@example.com")
    mock_db.get_product.return_value = Mock(id=1, price=10.0, inventory=100)
    mock_payment.charge.return_value = Mock(success=True, transaction_id="txn_123")
    mock_email.send.return_value = True
    mock_inventory.reduce.return_value = True
    mock_analytics.track.return_value = True
    
    # Create processor with all mocks
    processor = OrderProcessor(
        db=mock_db,
        payment=mock_payment,
        email=mock_email,
        inventory=mock_inventory,
        analytics=mock_analytics
    )
    
    # Process order
    result = processor.process_order({
        'user_id': 1,
        'items': [{'product_id': 1, 'quantity': 2}]
    })
    
    # Verify mocks were called
    assert mock_db.get_user.called
    assert mock_payment.charge.called
    assert mock_email.send.called
    assert result.success
```

**Problem**: This test passes, but it doesn't test anything meaningful. The entire business logic is mocked away. The test verifies that mocks were called, not that the code works correctly.

**What this test doesn't catch**:
- Incorrect price calculation
- Wrong tax rate
- Inventory not actually reduced
- Email sent with wrong content
- Payment charged wrong amount

**Better: Mock only external dependencies**:

```python
# tests/test_order_processor_better.py
def test_process_order_calculates_correct_total(db_session, mock_payment_gateway, mock_email_service):
    """Test that order processing calculates the correct total."""
    # Real database, real business logic
    user = create_user(db_session, email="test@example.com")
    product = create_product(db_session, name="Widget", price=10.0, inventory=100)
    
    # Mock only external services
    mock_payment_gateway.charge.return_value = {'transaction_id': 'txn_123', 'success': True}
    mock_email_service.send.return_value = True
    
    processor = OrderProcessor(
        db=db_session,
        payment_gateway=mock_payment_gateway,
        email_service=mock_email_service
    )
    
    result = processor.process_order({
        'user_id': user.id,
        'items': [{'product_id': product.id, 'quantity': 2}],
        'shipping_state': 'CA'
    })
    
    # Verify actual business logic
    assert result.success
    assert result.subtotal == 20.0  # 2 * $10
    assert result.tax == 1.45  # 7.25% of $20
    assert result.total == 21.45
    
    # Verify external services called correctly
    mock_payment_gateway.charge.assert_called_once_with(
        amount=21.45,
        user_id=user.id
    )
    
    # Verify inventory was actually reduced
    db_session.refresh(product)
    assert product.inventory == 98
```

**Improvements**:
- Real business logic is tested
- Only external dependencies are mocked
- Test catches calculation errors
- Test verifies side effects (inventory reduction)

### When to Mock vs. When to Use Real Objects

**Mock**:
- External APIs (payment gateways, email services, SMS)
- Slow operations (network calls, file I/O)
- Non-deterministic behavior (random number generation, current time)
- Third-party services you don't control

**Use real objects**:
- Business logic
- Data transformations
- Calculations
- Internal services you control
- Database operations (use test database or transactions)

### The Mocking Spectrum

| Level | What to Mock | What to Test | Example |
|-------|--------------|--------------|---------|
| Unit | Everything except the function under test | Single function logic | `test_calculate_tax()` |
| Integration | External services only | Multiple components working together | `test_order_processing_flow()` |
| E2E | Nothing (or minimal) | Entire system | `test_complete_checkout_flow()` |

### Signs You're Over-Mocking

**1. Tests pass but production fails**:

If your tests are green but users report bugs, you're probably mocking too much.

**2. Tests are harder to write than the code**:

If setting up mocks takes more code than the function being tested, you're over-mocking.

**3. Tests don't fail when you break the code**:

```python
# Break the code
def calculate_total(items):
    return 0  # Always return 0 (obviously wrong)

# Test still passes (because everything is mocked)
def test_calculate_total():
    mock_items = Mock()
    mock_items.sum.return_value = 100
    result = calculate_total(mock_items)
    assert result == mock_items.sum.return_value  # Passes!
```

**4. You're mocking your own code**:

If you're mocking functions in the same module you're testing, you're testing the wrong thing.

### How to Reduce Mocking

**1. Use dependency injection**:

```python
# Bad: Hard to test without mocking
class OrderProcessor:
    def __init__(self):
        self.payment_gateway = PaymentGateway()  # Hard-coded dependency
        self.email_service = EmailService()      # Hard-coded dependency
    
    def process_order(self, order):
        # Uses hard-coded dependencies
        pass

# Good: Easy to test with real or mock dependencies
class OrderProcessor:
    def __init__(self, payment_gateway, email_service):
        self.payment_gateway = payment_gateway
        self.email_service = email_service
    
    def process_order(self, order):
        # Uses injected dependencies
        pass
```

**2. Use test doubles instead of mocks**:

```python
# Instead of mocking, create a test implementation
class FakePaymentGateway:
    """Test double for payment gateway."""
    
    def __init__(self):
        self.charges = []
    
    def charge(self, amount, user_id):
        self.charges.append({'amount': amount, 'user_id': user_id})
        return {'transaction_id': f'txn_{len(self.charges)}', 'success': True}

def test_process_order_with_fake_gateway():
    """Test with fake implementation instead of mock."""
    fake_gateway = FakePaymentGateway()
    processor = OrderProcessor(payment_gateway=fake_gateway)
    
    processor.process_order({'user_id': 1, 'items': [...]})
    
    # Verify using the fake's state
    assert len(fake_gateway.charges) == 1
    assert fake_gateway.charges[0]['amount'] == 21.45
```

**3. Use integration tests for complex interactions**:

Instead of mocking everything in a unit test, write an integration test that uses real components.

## Testing Implementation Instead of Behavior

## The Implementation Testing Trap

Tests should verify **what** the code does, not **how** it does it. Testing implementation details makes tests brittle—they break when you refactor, even if behavior is unchanged.

**Bad: Testing implementation**:

```python
# src/user_service.py
class UserService:
    def __init__(self, db):
        self.db = db
        self._cache = {}
    
    def get_user(self, user_id):
        # Check cache first
        if user_id in self._cache:
            return self._cache[user_id]
        
        # Fetch from database
        user = self.db.query(User).filter_by(id=user_id).first()
        
        # Store in cache
        self._cache[user_id] = user
        
        return user

# tests/test_user_service_implementation.py (BAD)
def test_get_user_checks_cache_first():
    """Test that get_user checks cache before database."""
    db = Mock()
    service = UserService(db)
    
    # Verify cache is checked
    service._cache[1] = Mock(id=1, username="alice")
    user = service.get_user(1)
    
    # Test implementation detail: cache was checked
    assert db.query.call_count == 0  # Database not queried
    assert user.username == "alice"

def test_get_user_stores_in_cache():
    """Test that get_user stores result in cache."""
    db = Mock()
    db.query.return_value.filter_by.return_value.first.return_value = Mock(id=1)
    service = UserService(db)
    
    service.get_user(1)
    
    # Test implementation detail: cache was updated
    assert 1 in service._cache
```

**Problem**: These tests break if you change the caching implementation, even if the behavior is identical. For example, if you switch from a dict to Redis, these tests fail.

**Good: Testing behavior**:

```python
# tests/test_user_service_behavior.py (GOOD)
def test_get_user_returns_user_by_id(db_session):
    """Test that get_user returns the correct user."""
    # Setup
    user = create_user(db_session, id=1, username="alice")
    service = UserService(db_session)
    
    # Act
    result = service.get_user(1)
    
    # Assert behavior, not implementation
    assert result.id == 1
    assert result.username == "alice"

def test_get_user_returns_none_for_nonexistent_user(db_session):
    """Test that get_user returns None for nonexistent user."""
    service = UserService(db_session)
    
    result = service.get_user(999)
    
    assert result is None

def test_get_user_is_efficient_with_repeated_calls(db_session):
    """Test that repeated calls don't cause excessive database queries."""
    user = create_user(db_session, id=1, username="alice")
    service = UserService(db_session)
    
    # Make multiple calls
    result1 = service.get_user(1)
    result2 = service.get_user(1)
    result3 = service.get_user(1)
    
    # Verify behavior: all calls return the same user
    assert result1.id == result2.id == result3.id == 1
    
    # Optional: Verify efficiency (but not implementation)
    # This is acceptable because it tests a performance characteristic
    query_count = count_queries(db_session)
    assert query_count <= 1  # At most one query, regardless of implementation
```

**Improvements**:
- Tests verify what the code does (returns correct user)
- Tests don't care how caching is implemented
- Tests remain valid if you switch from dict to Redis
- Tests focus on observable behavior

### Examples of Implementation vs. Behavior

**Implementation testing** (brittle):

```python
# Testing that a specific method is called
def test_process_order_calls_validate():
    processor = OrderProcessor()
    with patch.object(processor, '_validate_order') as mock_validate:
        processor.process_order(order)
        mock_validate.assert_called_once()

# Testing internal state
def test_order_processor_sets_internal_flag():
    processor = OrderProcessor()
    processor.process_order(order)
    assert processor._processing_flag is True

# Testing method call order
def test_methods_called_in_correct_order():
    processor = OrderProcessor()
    with patch.object(processor, 'step1') as mock1, \
         patch.object(processor, 'step2') as mock2:
        processor.process_order(order)
        assert mock1.called
        assert mock2.called
        assert mock1.call_args < mock2.call_args  # Order matters
```

**Behavior testing** (robust):

```python
# Testing observable outcomes
def test_process_order_marks_order_as_completed():
    processor = OrderProcessor()
    result = processor.process_order(order)
    assert result.status == 'completed'

# Testing side effects
def test_process_order_reduces_inventory():
    processor = OrderProcessor()
    original_inventory = product.inventory
    processor.process_order(order)
    assert product.inventory == original_inventory - order.quantity

# Testing error conditions
def test_process_order_with_invalid_payment_raises_error():
    processor = OrderProcessor()
    with pytest.raises(PaymentError):
        processor.process_order(order_with_invalid_payment)
```

### When Implementation Testing Is Acceptable

**1. Testing performance characteristics**:

```python
def test_search_uses_index_for_performance():
    """Test that search uses database index (performance requirement)."""
    with assert_query_uses_index('users', 'username_idx'):
        search_users(username="alice")
```

**2. Testing security requirements**:

```python
def test_password_is_hashed_before_storage():
    """Test that passwords are hashed (security requirement)."""
    user = create_user(username="alice", password="plaintext")
    
    # Verify password is not stored in plaintext
    assert user.password_hash != "plaintext"
    assert len(user.password_hash) == 60  # bcrypt hash length
```

**3. Testing that expensive operations are avoided**:

```python
def test_get_user_does_not_query_database_on_cache_hit():
    """Test that cached users don't cause database queries."""
    service = UserService(db_session)
    
    # First call: cache miss
    service.get_user(1)
    
    # Second call: should use cache
    with assert_no_database_queries():
        service.get_user(1)
```

### How to Refactor Implementation Tests

**Step 1: Identify what behavior the test is trying to verify**:

```python
# Implementation test
def test_order_processor_calls_payment_gateway():
    processor = OrderProcessor()
    with patch.object(processor, 'payment_gateway') as mock:
        processor.process_order(order)
        mock.charge.assert_called_once()
```

**Question**: What behavior does this test verify?
**Answer**: That payment is processed when an order is processed.

**Step 2: Rewrite to test the behavior**:

```python
# Behavior test
def test_process_order_charges_payment():
    processor = OrderProcessor(payment_gateway=FakePaymentGateway())
    
    result = processor.process_order(order)
    
    assert result.payment_status == 'paid'
    assert result.transaction_id is not None
```

**Step 3: Verify the test still catches bugs**:

Change the code to break the behavior:

```python
def process_order(self, order):
    # Bug: Forgot to process payment
    # self.payment_gateway.charge(order.total)
    return OrderResult(status='completed', payment_status='paid')
```

The behavior test should fail. If it doesn't, the test isn't testing the right thing.

## Flaky Tests and Timing Issues

## The Flaky Test Problem

Flaky tests pass sometimes and fail sometimes, without any code changes. They erode trust in the test suite and waste developer time.

**Common causes of flakiness**:

1. **Race conditions**
2. **Timing dependencies**
3. **Non-deterministic behavior**
4. **External dependencies**
5. **Test order dependencies**

### Cause 1: Race Conditions

**Bad: Test with race condition**:

```python
# tests/test_async_processing.py (FLAKY)
import asyncio

async def test_process_items_concurrently():
    """Test concurrent item processing (flaky)."""
    results = []
    
    async def process_item(item):
        await asyncio.sleep(0.01)  # Simulate async work
        results.append(item)
    
    items = [1, 2, 3, 4, 5]
    await asyncio.gather(*[process_item(item) for item in items])
    
    # Flaky: Order is non-deterministic
    assert results == [1, 2, 3, 4, 5]
```

**Why it's flaky**: The order in which async tasks complete is non-deterministic. Sometimes the test passes, sometimes it fails.

**Fix: Test behavior, not order**:

```python
# tests/test_async_processing.py (FIXED)
async def test_process_items_concurrently():
    """Test concurrent item processing."""
    results = []
    
    async def process_item(item):
        await asyncio.sleep(0.01)
        results.append(item)
    
    items = [1, 2, 3, 4, 5]
    await asyncio.gather(*[process_item(item) for item in items])
    
    # Test behavior: all items processed (order doesn't matter)
    assert sorted(results) == [1, 2, 3, 4, 5]
    assert len(results) == 5
```

### Cause 2: Timing Dependencies

**Bad: Test with timing dependency**:

```python
# tests/test_cache_expiration.py (FLAKY)
import time

def test_cache_expires_after_timeout():
    """Test that cache entries expire (flaky)."""
    cache = Cache(ttl=1.0)  # 1 second TTL
    
    cache.set('key', 'value')
    assert cache.get('key') == 'value'
    
    # Wait for expiration
    time.sleep(1.0)
    
    # Flaky: Timing is imprecise
    assert cache.get('key') is None
```

**Why it's flaky**: `time.sleep(1.0)` doesn't guarantee exactly 1 second. On a slow CI server, it might be 0.99 seconds, and the test fails.

**Fix: Use explicit time control**:

```python
# tests/test_cache_expiration.py (FIXED)
def test_cache_expires_after_timeout(freezer):
    """Test that cache entries expire."""
    cache = Cache(ttl=1.0)
    
    cache.set('key', 'value')
    assert cache.get('key') == 'value'
    
    # Fast-forward time (no actual waiting)
    freezer.move_to(datetime.now() + timedelta(seconds=1.1))
    
    # Deterministic: Time is controlled
    assert cache.get('key') is None
```

**Alternative: Use timeouts with margin**:

```python
def test_cache_expires_after_timeout():
    """Test that cache entries expire."""
    cache = Cache(ttl=1.0)
    
    cache.set('key', 'value')
    assert cache.get('key') == 'value'
    
    # Wait longer than TTL with margin
    time.sleep(1.5)
    
    # More reliable with margin
    assert cache.get('key') is None
```

### Cause 3: Non-Deterministic Behavior

**Bad: Test with random behavior**:

```python
# tests/test_random_selection.py (FLAKY)
import random

def test_select_random_item():
    """Test random item selection (flaky)."""
    items = [1, 2, 3, 4, 5]
    
    selected = select_random_item(items)
    
    # Flaky: Random selection
    assert selected == 1
```

**Fix: Control randomness**:

```python
# tests/test_random_selection.py (FIXED)
def test_select_random_item():
    """Test random item selection."""
    items = [1, 2, 3, 4, 5]
    
    # Seed random number generator for deterministic behavior
    random.seed(42)
    
    selected = select_random_item(items)
    
    # Deterministic with seeded RNG
    assert selected == 3  # Known result with seed 42

def test_select_random_item_returns_valid_item():
    """Test that random selection returns a valid item."""
    items = [1, 2, 3, 4, 5]
    
    selected = select_random_item(items)
    
    # Test behavior, not specific value
    assert selected in items
```

### Cause 4: External Dependencies

**Bad: Test with external dependency**:

```python
# tests/test_api_client.py (FLAKY)
def test_fetch_user_data():
    """Test fetching user data from API (flaky)."""
    client = APIClient("https://api.example.com")
    
    # Flaky: Depends on external API being available
    user = client.get_user(1)
    
    assert user['username'] == 'alice'
```

**Why it's flaky**: External API might be down, slow, or return different data.

**Fix: Mock external dependencies**:

```python
# tests/test_api_client.py (FIXED)
def test_fetch_user_data(mock_requests):
    """Test fetching user data from API."""
    mock_requests.get.return_value.json.return_value = {
        'id': 1,
        'username': 'alice'
    }
    
    client = APIClient("https://api.example.com")
    user = client.get_user(1)
    
    assert user['username'] == 'alice'
```

### Cause 5: Test Order Dependencies

**Bad: Tests depend on execution order**:

```python
# tests/test_user_lifecycle.py (FLAKY)
_user_id = None

def test_create_user():
    global _user_id
    user = create_user(username="alice")
    _user_id = user.id

def test_update_user():
    global _user_id
    # Flaky: Depends on test_create_user running first
    update_user(_user_id, email="newemail@example.com")
```

**Fix: Make tests independent** (see section 20.3.4).

### Detecting Flaky Tests

**Run tests multiple times**:

```bash
# Run each test 100 times
$ pytest --count=100 tests/test_flaky.py
```

**Run tests in random order**:

```bash
$ pytest --random-order tests/
```

**Run tests in parallel**:

```bash
$ pytest -n auto tests/
```

If tests fail with any of these approaches but pass normally, they're flaky.

### Quarantining Flaky Tests

While fixing flaky tests, quarantine them so they don't block CI:

```python
# conftest.py
def pytest_configure(config):
    config.addinivalue_line("markers", "flaky: known flaky test (quarantined)")

# tests/test_flaky.py
@pytest.mark.flaky
def test_sometimes_fails():
    """This test is flaky and needs fixing."""
    pass
```

Run tests excluding flaky ones:

```bash
$ pytest -m "not flaky"
```

Run only flaky tests to debug them:

```bash
$ pytest -m flaky --count=100
```

## Ignoring Test Maintenance

## The Technical Debt Problem

Tests are code. Like all code, they need maintenance. Ignoring test maintenance leads to:
- Slow test suites
- Brittle tests that break on refactoring
- Duplicate test code
- Outdated tests that don't reflect current behavior

### Symptom 1: Slow Test Suite

**Problem**: Test suite takes 45 minutes to run, slowing down development.

**Diagnosis**:

```bash
# Find slowest tests
$ pytest --durations=20
```

**Output**:

```text
======================== slowest 20 durations ========================
45.23s call     tests/test_report_generation.py::test_generate_annual_report
32.11s call     tests/test_email.py::test_send_bulk_emails
28.45s call     tests/test_data_import.py::test_import_large_dataset
12.34s call     tests/test_api_integration.py::test_full_api_flow
...
```

**Solutions**:

1. **Optimize slow tests**:

```python
# Before: Slow test
def test_generate_annual_report():
    """Generate report for entire year (45 seconds)."""
    report = generate_report(
        start_date=datetime(2023, 1, 1),
        end_date=datetime(2023, 12, 31)
    )
    assert report.total_revenue > 0

# After: Fast test
def test_generate_report_calculates_revenue():
    """Generate report for one day (0.5 seconds)."""
    report = generate_report(
        start_date=datetime(2023, 1, 1),
        end_date=datetime(2023, 1, 1)
    )
    assert report.total_revenue > 0
```

2. **Mark slow tests and run them separately**:

```python
@pytest.mark.slow
def test_generate_annual_report():
    """Generate report for entire year."""
    pass
```

```bash
# Fast tests (run constantly)
$ pytest -m "not slow"

# Slow tests (run nightly)
$ pytest -m slow
```

3. **Use fixtures to share expensive setup**:

```python
# Before: Each test creates database
def test_query_users():
    db = create_test_database()  # 5 seconds
    # ... test

def test_query_orders():
    db = create_test_database()  # 5 seconds
    # ... test

# After: Share database across tests
@pytest.fixture(scope="module")
def test_database():
    db = create_test_database()  # 5 seconds once
    yield db
    db.cleanup()

def test_query_users(test_database):
    # ... test (no setup time)

def test_query_orders(test_database):
    # ... test (no setup time)
```

### Symptom 2: Duplicate Test Code

**Problem**: Same test logic repeated across multiple files.

**Example**:

```python
# tests/test_user_api.py
def test_create_user_api():
    user = create_user(username="alice", email="alice@example.com")
    assert user.id is not None
    assert user.username == "alice"
    assert user.email == "alice@example.com"

# tests/test_user_service.py
def test_create_user_service():
    user = create_user(username="alice", email="alice@example.com")
    assert user.id is not None
    assert user.username == "alice"
    assert user.email == "alice@example.com"

# tests/test_user_model.py
def test_create_user_model():
    user = create_user(username="alice", email="alice@example.com")
    assert user.id is not None
    assert user.username == "alice"
    assert user.email == "alice@example.com"
```

**Solution: Extract common assertions**:

```python
# tests/helpers.py
def assert_valid_user(user, expected_username, expected_email):
    """Assert that user object is valid."""
    assert user.id is not None
    assert user.username == expected_username
    assert user.email == expected_email

# tests/test_user_api.py
def test_create_user_api():
    user = create_user(username="alice", email="alice@example.com")
    assert_valid_user(user, "alice", "alice@example.com")

# tests/test_user_service.py
def test_create_user_service():
    user = create_user(username="alice", email="alice@example.com")
    assert_valid_user(user, "alice", "alice@example.com")
```

### Symptom 3: Outdated Tests

**Problem**: Tests pass but don't reflect current behavior.

**Example**:

```python
# tests/test_order_processing.py (written 2 years ago)
def test_process_order():
    """Test order processing."""
    order = create_order(items=[{'product_id': 1, 'quantity': 2}])
    
    result = process_order(order)
    
    # Test written before we added tax calculation
    assert result.total == 20.0  # Wrong: Doesn't include tax
```

**Solution: Regular test review**:

1. **Review tests when code changes**:

```python
# When adding tax calculation, update tests
def test_process_order():
    """Test order processing with tax."""
    order = create_order(
        items=[{'product_id': 1, 'quantity': 2}],
        shipping_state='CA'
    )
    
    result = process_order(order)
    
    # Updated to include tax
    assert result.subtotal == 20.0
    assert result.tax == 1.45  # 7.25% of $20
    assert result.total == 21.45
```

2. **Add tests for new features**:

When adding a feature, add tests for it. Don't just modify existing tests.

3. **Remove obsolete tests**:

If a feature is removed, remove its tests. Don't leave dead code.

### Symptom 4: Brittle Tests

**Problem**: Tests break when you refactor, even though behavior is unchanged.

**Example**:

```python
# Before refactoring
class OrderProcessor:
    def process(self, order):
        self._validate(order)
        self._calculate_total(order)
        self._charge_payment(order)
        return order

# Brittle test
def test_process_order():
    processor = OrderProcessor()
    with patch.object(processor, '_validate') as mock_validate, \
         patch.object(processor, '_calculate_total') as mock_calc, \
         patch.object(processor, '_charge_payment') as mock_charge:
        
        processor.process(order)
        
        # Tests implementation details
        mock_validate.assert_called_once()
        mock_calc.assert_called_once()
        mock_charge.assert_called_once()
```

**After refactoring** (behavior unchanged):

```python
class OrderProcessor:
    def process(self, order):
        # Refactored: Combined validation and calculation
        self._prepare_order(order)
        self._charge_payment(order)
        return order
    
    def _prepare_order(self, order):
        self._validate(order)
        self._calculate_total(order)
```

**Test breaks** even though behavior is identical.

**Solution: Test behavior, not implementation** (see section 20.4.2).

### Test Maintenance Checklist

**Weekly**:
- [ ] Review failed tests in CI
- [ ] Fix or quarantine flaky tests
- [ ] Check test execution time

**Monthly**:
- [ ] Review slowest tests
- [ ] Identify and remove duplicate test code
- [ ] Update tests for new features

**Quarterly**:
- [ ] Review test coverage
- [ ] Remove obsolete tests
- [ ] Refactor brittle tests
- [ ] Update test documentation

## Coverage Theater

## The Coverage Illusion

High code coverage doesn't guarantee good tests. You can have 100% coverage with tests that don't actually verify anything.

**Bad: High coverage, low value**:

```python
# src/calculator.py
def calculate_total(items, tax_rate=0.0725):
    """Calculate order total with tax."""
    subtotal = sum(item['price'] * item['quantity'] for item in items)
    tax = subtotal * tax_rate
    return subtotal + tax

# tests/test_calculator.py (BAD)
def test_calculate_total():
    """Test calculate_total function."""
    # This test achieves 100% coverage but doesn't verify correctness
    result = calculate_total([{'price': 10.0, 'quantity': 2}])
    assert result is not None  # Useless assertion
```

**Coverage report**:

```text
src/calculator.py    100%    (all lines covered)
```

**But the test doesn't verify**:
- Correct calculation
- Tax rate application
- Edge cases (empty list, zero prices, etc.)

**Good: Meaningful tests**:

```python
# tests/test_calculator.py (GOOD)
def test_calculate_total_with_single_item():
    """Test total calculation with one item."""
    items = [{'price': 10.0, 'quantity': 2}]
    
    result = calculate_total(items, tax_rate=0.0725)
    
    assert result == 21.45  # 20.00 + 1.45 tax

def test_calculate_total_with_multiple_items():
    """Test total calculation with multiple items."""
    items = [
        {'price': 10.0, 'quantity': 2},
        {'price': 5.0, 'quantity': 3}
    ]
    
    result = calculate_total(items, tax_rate=0.0725)
    
    assert result == 37.54  # 35.00 + 2.54 tax

def test_calculate_total_with_zero_tax():
    """Test total calculation with no tax."""
    items = [{'price': 10.0, 'quantity': 2}]
    
    result = calculate_total(items, tax_rate=0.0)
    
    assert result == 20.0

def test_calculate_total_with_empty_list():
    """Test total calculation with no items."""
    result = calculate_total([], tax_rate=0.0725)
    
    assert result == 0.0
```

**Coverage report** (same 100%, but meaningful):

```text
src/calculator.py    100%    (all lines covered with meaningful tests)
```

### Signs of Coverage Theater

**1. Tests that don't assert anything meaningful**:

```python
# Bad: Achieves coverage but doesn't test anything
def test_function_runs():
    result = complex_function(input_data)
    assert result is not None  # Useless
    assert isinstance(result, dict)  # Barely useful
```

**2. Tests that only test happy paths**:

```python
# Bad: Only tests success case
def test_divide():
    assert divide(10, 2) == 5

# Missing: What about divide(10, 0)?
```

**3. Tests that mock everything**:

```python
# Bad: Mocks away all logic
def test_process_order():
    mock_everything = Mock()
    mock_everything.process.return_value = Mock(success=True)
    
    result = process_order(mock_everything)
    
    assert result.success  # Tests nothing
```

**4. Coverage goals without quality goals**:

```text
# Bad: Focus on coverage percentage
"We need 90% coverage!"

# Good: Focus on test quality
"We need tests that catch bugs and document behavior."
```

### Meaningful Coverage Metrics

**Instead of raw coverage percentage, track**:

1. **Mutation testing score**: How many bugs do your tests catch?

```bash
$ pip install mutmut
$ mutmut run
```

**Output**:

```text
Mutations: 100
Killed: 85 (85%)
Survived: 15 (15%)
```

**Interpretation**: Your tests catch 85% of introduced bugs. The 15% that survive indicate gaps in test quality.

2. **Branch coverage**: Are all code paths tested?

```bash
$ pytest --cov=src --cov-report=term-missing --cov-branch
```

**Output**:

```text
src/calculator.py    85%    (missing branches: 23->25, 30->32)
```

**Interpretation**: Lines 23-25 and 30-32 have untested branches (e.g., if/else conditions).

3. **Critical path coverage**: Are the most important features tested?

```python
# Mark critical paths
@pytest.mark.critical
def test_payment_processing():
    """Test payment processing (critical path)."""
    pass

@pytest.mark.critical
def test_user_authentication():
    """Test user authentication (critical path)."""
    pass
```

**Verify critical paths are covered**:

```bash
$ pytest -m critical --cov=src --cov-report=term
```

### Using Coverage Correctly

**1. Use coverage to find untested code**:

```bash
$ pytest --cov=src --cov-report=html
$ open htmlcov/index.html
```

Look for red (untested) lines. Ask: "Should this code be tested?"

**2. Don't aim for 100% coverage**:

Some code doesn't need tests:
- Trivial getters/setters
- Framework boilerplate
- Code that's tested indirectly

**3. Focus on coverage of critical code**:

```python
# High-value tests
def test_payment_processing():
    """Test payment processing (critical)."""
    pass

def test_data_validation():
    """Test data validation (prevents corruption)."""
    pass

# Low-value tests
def test_getter():
    """Test simple getter (low value)."""
    user = User(name="Alice")
    assert user.name == "Alice"  # Trivial
```

**4. Use coverage as a guide, not a goal**:

```text
# Bad: Coverage-driven development
"We need 90% coverage, so let's write tests until we hit 90%."

# Good: Quality-driven development
"Let's write tests that verify critical behavior and catch bugs.
 Coverage will naturally increase as a side effect."
```

## Tests That Pass When They Shouldn't

## The False Positive Problem

The worst kind of test is one that passes when the code is broken. These tests give false confidence and hide bugs.

**Example 1: Incorrect assertion**:

```python
# tests/test_user_creation.py (BAD)
def test_create_user_assigns_id():
    """Test that creating a user assigns an ID."""
    user = create_user(username="alice")
    
    # Bug: This always passes
    assert user.id or True  # Always True!
```

**The code could be completely broken, but the test passes.**

**Fix**:

```python
def test_create_user_assigns_id():
    """Test that creating a user assigns an ID."""
    user = create_user(username="alice")
    
    assert user.id is not None
    assert isinstance(user.id, int)
    assert user.id > 0
```

**Example 2: Catching wrong exception**:

```python
# tests/test_validation.py (BAD)
def test_invalid_email_raises_error():
    """Test that invalid email raises ValidationError."""
    with pytest.raises(Exception):  # Too broad!
        create_user(username="alice", email="invalid")
```

**Problem**: This test passes if ANY exception is raised, not just ValidationError. If the code raises KeyError or AttributeError, the test still passes.

**Fix**:

```python
def test_invalid_email_raises_validation_error():
    """Test that invalid email raises ValidationError."""
    with pytest.raises(ValidationError, match="Invalid email"):
        create_user(username="alice", email="invalid")
```

**Example 3: Mock that doesn't match reality**:

```python
# tests/test_payment.py (BAD)
def test_process_payment():
    """Test payment processing."""
    mock_gateway = Mock()
    # Mock returns success, but real gateway returns different structure
    mock_gateway.charge.return_value = {'success': True}
    
    processor = PaymentProcessor(gateway=mock_gateway)
    result = processor.process_payment(amount=100.0)
    
    assert result.success
```

**Problem**: Real payment gateway returns `{'status': 'completed', 'transaction_id': '...'}`, not `{'success': True}`. Test passes, but production code breaks.

**Fix: Use realistic mocks**:

```python
def test_process_payment():
    """Test payment processing."""
    mock_gateway = Mock()
    # Mock matches real gateway response
    mock_gateway.charge.return_value = {
        'status': 'completed',
        'transaction_id': 'txn_123',
        'amount': 100.0
    }
    
    processor = PaymentProcessor(gateway=mock_gateway)
    result = processor.process_payment(amount=100.0)
    
    assert result.status == 'completed'
    assert result.transaction_id == 'txn_123'
```

**Example 4: Test that doesn't actually call the code**:

```python
# tests/test_calculator.py (BAD)
def test_calculate_total():
    """Test total calculation."""
    items = [{'price': 10.0, 'quantity': 2}]
    
    # Bug: Forgot to call the function!
    # result = calculate_total(items)
    
    # This assertion always passes
    assert True
```

**Fix: Actually call the function**:

```python
def test_calculate_total():
    """Test total calculation."""
    items = [{'price': 10.0, 'quantity': 2}]
    
    result = calculate_total(items)
    
    assert result == 21.45
```

### Detecting False Positives

**1. Mutation testing**:

Introduce bugs and verify tests fail:

```bash
$ pip install mutmut
$ mutmut run
```

**Output**:

```text
Mutation survived: Changed 'return subtotal + tax' to 'return subtotal - tax'
Test test_calculate_total still passed!
```

**Interpretation**: The test doesn't catch this bug. It's a false positive.

**2. Temporarily break the code**:

Manually introduce a bug and verify the test fails:

```python
# src/calculator.py (temporarily broken)
def calculate_total(items, tax_rate=0.0725):
    subtotal = sum(item['price'] * item['quantity'] for item in items)
    tax = subtotal * tax_rate
    return 0  # Bug: Always return 0
```

Run the test:

```bash
$ pytest tests/test_calculator.py::test_calculate_total
```

**If the test passes, it's a false positive.**

**3. Review test assertions**:

Look for weak assertions:

```python
# Weak assertions (likely false positives)
assert result  # Too vague
assert result is not None  # Barely useful
assert isinstance(result, dict)  # Doesn't verify content
assert len(result) > 0  # Doesn't verify correctness

# Strong assertions
assert result == expected_value
assert result.status == 'completed'
assert result.total == 21.45
```

### Preventing False Positives

**1. Write the test first (TDD)**:

If you write the test before the code, you'll see it fail first:

```python
# Step 1: Write test (it fails because code doesn't exist)
def test_calculate_total():
    result = calculate_total([{'price': 10.0, 'quantity': 2}])
    assert result == 21.45

# Step 2: Write code to make it pass
def calculate_total(items, tax_rate=0.0725):
    subtotal = sum(item['price'] * item['quantity'] for item in items)
    tax = subtotal * tax_rate
    return subtotal + tax

# Step 3: Verify test passes
```

**2. Use specific assertions**:

```python
# Bad: Vague assertion
assert result

# Good: Specific assertion
assert result.status == 'completed'
assert result.transaction_id.startswith('txn_')
assert result.amount == 100.0
```

**3. Test error cases**:

Don't just test success. Test failures too:

```python
def test_divide_by_zero_raises_error():
    """Test that division by zero raises an error."""
    with pytest.raises(ZeroDivisionError):
        divide(10, 0)

def test_invalid_input_raises_validation_error():
    """Test that invalid input raises ValidationError."""
    with pytest.raises(ValidationError, match="Invalid email"):
        create_user(email="invalid")
```

**4. Use pytest's strict mode**:

```ini
# pytest.ini
[pytest]
# Fail on warnings
filterwarnings = error

# Fail on unraisable exceptions
pythonwarnings = error
```

This catches issues that might otherwise be silently ignored.

## Where to Go From Here

You've completed the journey from pytest beginner to advanced practitioner. You've learned:

- Core pytest features (fixtures, parametrization, markers)
- Advanced techniques (mocking, async testing, property-based testing)
- Real-world patterns (legacy code testing, microservices, large codebases)
- Best practices and common pitfalls

## Continuing Your Testing Journey

### 1. Practice Deliberately

**Apply what you've learned**:
- Refactor one test file in your current project using best practices
- Add property-based tests to a critical function
- Set up contract testing for a microservice
- Implement test impact analysis for faster feedback

**Start small**: Don't try to apply everything at once. Pick one technique per week and master it.

### 2. Contribute to Open Source

**Learn from others**:
- Read test suites of popular Python projects (Django, Flask, Requests)
- Contribute tests to open source projects
- Review pull requests that add or modify tests

**Projects with excellent test suites**:
- **pytest itself**: https://github.com/pytest-dev/pytest
- **Requests**: https://github.com/psf/requests
- **Django**: https://github.com/django/django
- **FastAPI**: https://github.com/tiangolo/fastapi

### 3. Explore Advanced Topics

**Beyond this book**:

**Property-Based Testing**:
- Hypothesis documentation: https://hypothesis.readthedocs.io/
- "Property-Based Testing with PropEr, Erlang, and Elixir" by Fred Hebert

**Mutation Testing**:
- mutmut: https://github.com/boxed/mutmut
- cosmic-ray: https://github.com/sixty-north/cosmic-ray

**Contract Testing**:
- Pact documentation: https://docs.pact.io/
- Spring Cloud Contract (for polyglot systems)

**Performance Testing**:
- pytest-benchmark: https://pytest-benchmark.readthedocs.io/
- locust (load testing): https://locust.io/

### 4. Build Your Testing Toolkit

**Essential pytest plugins**:

```bash
# Core plugins
pip install pytest-cov          # Coverage reporting
pip install pytest-xdist        # Parallel execution
pip install pytest-asyncio      # Async test support
pip install pytest-mock         # Enhanced mocking

# Productivity plugins
pip install pytest-watch        # Auto-run tests
pip install pytest-sugar        # Better output
pip install pytest-html         # HTML reports

# Advanced plugins
pip install pytest-benchmark   # Performance testing
pip install hypothesis         # Property-based testing
pip install pytest-timeout     # Prevent hanging tests
pip install pytest-randomly    # Randomize test order
```

### 5. Stay Current

**Follow pytest development**:
- pytest changelog: https://docs.pytest.org/en/stable/changelog.html
- pytest blog: https://pytest.org/blog/
- pytest on Twitter: @pytestdotorg

**Join the community**:
- pytest Discord: https://discord.gg/pytest
- pytest mailing list: pytest-dev@python.org
- Stack Overflow: [pytest] tag

### 6. Teach Others

**The best way to master testing**:
- Write blog posts about testing techniques you've learned
- Give talks at local Python meetups
- Mentor junior developers on testing practices
- Conduct code reviews focused on test quality

**Teaching solidifies your understanding and helps the community.**

### 7. Measure Your Progress

**Track your testing maturity**:

**Level 1: Beginner**
- [ ] Can write basic test functions
- [ ] Understands assertions
- [ ] Can run tests with pytest

**Level 2: Intermediate**
- [ ] Uses fixtures effectively
- [ ] Writes parametrized tests
- [ ] Understands test organization
- [ ] Can mock external dependencies

**Level 3: Advanced**
- [ ] Writes property-based tests
- [ ] Implements contract testing
- [ ] Optimizes test suite performance
- [ ] Mentors others on testing

**Level 4: Expert**
- [ ] Designs testing strategies for large systems
- [ ] Contributes to pytest or plugins
- [ ] Writes about testing practices
- [ ] Influences team testing culture

## Final Thoughts

Testing is not just about finding bugs—it's about:
- **Confidence**: Knowing your code works
- **Documentation**: Tests describe how code should behave
- **Design**: Tests reveal design problems
- **Refactoring**: Tests enable safe changes
- **Communication**: Tests communicate intent to other developers

**The best tests are**:
- **Fast**: Run in milliseconds, not minutes
- **Isolated**: Independent of each other
- **Repeatable**: Same result every time
- **Self-validating**: Pass or fail clearly
- **Timely**: Written close to the code they test

**Remember**:
- Perfect is the enemy of good. Start with simple tests and improve over time.
- Tests are code. Apply the same quality standards to tests as to production code.
- Coverage is a tool, not a goal. Focus on meaningful tests, not percentages.
- Flaky tests are worse than no tests. Fix or remove them.
- Tests should help you move faster, not slower. If they don't, something is wrong.

**You now have the knowledge to write excellent tests. Go forth and test with confidence!**

## Cheat Sheet: Common Pytest Commands and Patterns

## Quick Reference

### Running Tests

```bash
# Run all tests
pytest

# Run tests in a specific file
pytest tests/test_user.py

# Run a specific test function
pytest tests/test_user.py::test_create_user

# Run tests matching a pattern
pytest -k "user and not delete"

# Run tests with a specific marker
pytest -m slow

# Run tests in parallel
pytest -n auto

# Run with verbose output
pytest -v

# Run with extra verbose output
pytest -vv

# Stop on first failure
pytest -x

# Run last failed tests
pytest --lf

# Run failed tests first, then others
pytest --ff

# Show local variables in tracebacks
pytest -l

# Drop into debugger on failure
pytest --pdb

# Run tests and show coverage
pytest --cov=src --cov-report=term-missing
```

### Writing Tests

```python
# Basic test
def test_example():
    assert 1 + 1 == 2

# Test with fixture
def test_with_fixture(db_session):
    user = create_user(db_session)
    assert user.id is not None

# Parametrized test
@pytest.mark.parametrize("input,expected", [
    (1, 2),
    (2, 4),
    (3, 6),
])
def test_double(input, expected):
    assert double(input) == expected

# Test exception
def test_raises_error():
    with pytest.raises(ValueError, match="Invalid"):
        raise ValueError("Invalid input")

# Test with multiple assertions
def test_user_creation():
    user = create_user(username="alice")
    assert user.username == "alice"
    assert user.id is not None
    assert user.created_at is not None

# Skip test
@pytest.mark.skip(reason="Not implemented yet")
def test_future_feature():
    pass

# Skip test conditionally
@pytest.mark.skipif(sys.version_info < (3, 8), reason="Requires Python 3.8+")
def test_new_feature():
    pass

# Expected failure
@pytest.mark.xfail(reason="Known bug")
def test_known_bug():
    assert buggy_function() == expected_value
```

### Fixtures

```python
# Basic fixture
@pytest.fixture
def user():
    return create_user(username="alice")

# Fixture with setup and teardown
@pytest.fixture
def db_session():
    session = create_session()
    yield session
    session.rollback()
    session.close()

# Fixture with scope
@pytest.fixture(scope="module")
def database():
    db = create_database()
    yield db
    db.drop()

# Fixture that uses another fixture
@pytest.fixture
def authenticated_user(user, auth_service):
    token = auth_service.authenticate(user)
    return user, token

# Factory fixture
@pytest.fixture
def user_factory(db_session):
    def _create_user(**kwargs):
        return create_user(db_session, **kwargs)
    return _create_user

# Autouse fixture
@pytest.fixture(autouse=True)
def reset_state():
    global_state.clear()
    yield
    global_state.clear()
```

### Mocking

```python
# Mock with unittest.mock
from unittest.mock import Mock, patch

# Create a mock
mock_service = Mock()
mock_service.get_user.return_value = {'id': 1, 'username': 'alice'}

# Patch a function
with patch('module.function') as mock_func:
    mock_func.return_value = 42
    result = call_function_that_uses_function()
    assert result == 42

# Patch as decorator
@patch('module.function')
def test_with_patch(mock_func):
    mock_func.return_value = 42
    result = call_function_that_uses_function()
    assert result == 42

# Mock side effects
mock_service.get_user.side_effect = [
    {'id': 1},
    {'id': 2},
    Exception("Error")
]

# Assert mock was called
mock_service.get_user.assert_called_once()
mock_service.get_user.assert_called_with(user_id=1)
mock_service.get_user.assert_called_once_with(user_id=1)

# Check call count
assert mock_service.get_user.call_count == 3
```

### Markers

```python
# Built-in markers
@pytest.mark.skip
@pytest.mark.skipif(condition, reason="...")
@pytest.mark.xfail
@pytest.mark.parametrize("arg", [1, 2, 3])

# Custom markers (define in pytest.ini)
@pytest.mark.slow
@pytest.mark.integration
@pytest.mark.unit
@pytest.mark.smoke

# Multiple markers
@pytest.mark.slow
@pytest.mark.integration
def test_complex_integration():
    pass
```

### Configuration (pytest.ini)

```ini
[pytest]
# Test discovery patterns
python_files = test_*.py *_test.py
python_classes = Test*
python_functions = test_*

# Add options to all test runs
addopts = -v --strict-markers --cov=src

# Markers
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    integration: marks tests as integration tests
    unit: marks tests as unit tests

# Minimum Python version
minversion = 3.8

# Test paths
testpaths = tests

# Ignore paths
norecursedirs = .git .tox dist build *.egg
```

### Common Patterns

```python
# Test database operations
@pytest.fixture
def db_session():
    session = create_session()
    yield session
    session.rollback()
    session.close()

def test_create_user(db_session):
    user = create_user(db_session, username="alice")
    assert user.id is not None

# Test async code
@pytest.mark.asyncio
async def test_async_function():
    result = await async_function()
    assert result == expected

# Test with temporary files
def test_file_operations(tmp_path):
    file_path = tmp_path / "test.txt"
    file_path.write_text("content")
    assert file_path.read_text() == "content"

# Test with environment variables
def test_with_env_var(monkeypatch):
    monkeypatch.setenv("API_KEY", "test_key")
    result = function_that_uses_env_var()
    assert result is not None

# Test with time control
def test_with_frozen_time(freezer):
    freezer.move_to("2024-01-01 12:00:00")
    result = function_that_uses_current_time()
    assert result.date() == date(2024, 1, 1)

# Test with captured output
def test_print_output(capsys):
    print("Hello, World!")
    captured = capsys.readouterr()
    assert "Hello" in captured.out
```

### Debugging Tests

```bash
# Show print statements
pytest -s

# Show local variables on failure
pytest -l

# Drop into debugger on failure
pytest --pdb

# Drop into debugger at start of test
pytest --trace

# Show full diff on assertion failure
pytest -vv

# Show test durations
pytest --durations=10

# Run with warnings
pytest -W error

# Verbose output with full tracebacks
pytest -vv --tb=long
```

### Performance Optimization

```bash
# Run tests in parallel
pytest -n auto

# Run only fast tests
pytest -m "not slow"

# Run with test impact analysis
pytest --lf --ff

# Profile test execution
pytest --durations=0

# Use pytest-xdist with load balancing
pytest -n auto --dist loadscope
```

### Coverage

```bash
# Basic coverage
pytest --cov=src

# Coverage with missing lines
pytest --cov=src --cov-report=term-missing

# HTML coverage report
pytest --cov=src --cov-report=html

# Coverage with branch coverage
pytest --cov=src --cov-branch

# Fail if coverage below threshold
pytest --cov=src --cov-fail-under=80
```

### CI/CD Integration

```yaml
# GitHub Actions
name: Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-python@v2
        with:
          python-version: 3.11
      - run: pip install -r requirements.txt
      - run: pytest --cov=src --cov-report=xml
      - uses: codecov/codecov-action@v2
```

### Quick Tips

**Test Organization**:
- One test file per source file: `src/user.py` → `tests/test_user.py`
- Group related tests in classes: `class TestUserCreation:`
- Use descriptive test names: `test_create_user_with_invalid_email_raises_error`

**Fixture Best Practices**:
- Use fixtures for setup, not test data
- Keep fixtures simple and focused
- Use factory fixtures for flexibility
- Document fixture purpose and usage

**Assertion Best Practices**:
- One logical assertion per test
- Use specific assertions: `assert x == 5`, not `assert x`
- Include helpful error messages: `assert x == 5, f"Expected 5, got {x}"`

**Mocking Best Practices**:
- Mock external dependencies only
- Use realistic mock data
- Verify mock calls when behavior matters
- Prefer test doubles over mocks when possible

**Performance Best Practices**:
- Mark slow tests: `@pytest.mark.slow`
- Use fixtures with appropriate scope
- Run tests in parallel: `pytest -n auto`
- Use test impact analysis: `pytest --lf --ff`

**Debugging Best Practices**:
- Use `pytest -vv` for detailed output
- Use `pytest --pdb` to debug failures
- Use `pytest -l` to see local variables
- Use `pytest -s` to see print statements

This cheat sheet covers the most common pytest commands and patterns. Keep it handy for quick reference!
