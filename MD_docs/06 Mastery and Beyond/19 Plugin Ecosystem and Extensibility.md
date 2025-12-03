# Chapter 19: Plugin Ecosystem and Extensibility

## Popular Pytest Plugins (pytest-html, pytest-xdist, pytest-sugar)

## The Plugin Ecosystem: Extending Pytest's Capabilities

Pytest's power comes not just from its core features, but from its extensible architecture. The plugin ecosystem transforms pytest from a testing framework into a complete testing platform. In this chapter, we'll explore how plugins work, how to use popular ones, and how to create your own.

Before diving into plugin creation, let's understand what plugins can do by examining three widely-used examples that solve real testing problems.

### The Reference Scenario: A Growing Test Suite

Let's establish a realistic testing scenario that will demonstrate why plugins matter. We're testing a web API client that handles user authentication, data retrieval, and error handling.

```python
# src/api_client.py
import requests
from typing import Optional, Dict, Any
import time

class APIClient:
    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url
        self.api_key = api_key
        self.session = requests.Session()
        self.session.headers.update({"Authorization": f"Bearer {api_key}"})
    
    def get_user(self, user_id: int) -> Dict[str, Any]:
        """Fetch user data from the API."""
        response = self.session.get(f"{self.base_url}/users/{user_id}")
        response.raise_for_status()
        return response.json()
    
    def create_user(self, username: str, email: str) -> Dict[str, Any]:
        """Create a new user."""
        response = self.session.post(
            f"{self.base_url}/users",
            json={"username": username, "email": email}
        )
        response.raise_for_status()
        return response.json()
    
    def update_user(self, user_id: int, data: Dict[str, Any]) -> Dict[str, Any]:
        """Update user data."""
        response = self.session.patch(
            f"{self.base_url}/users/{user_id}",
            json=data
        )
        response.raise_for_status()
        return response.json()
    
    def delete_user(self, user_id: int) -> bool:
        """Delete a user."""
        response = self.session.delete(f"{self.base_url}/users/{user_id}")
        response.raise_for_status()
        return True
    
    def search_users(self, query: str, limit: int = 10) -> list:
        """Search for users matching query."""
        time.sleep(0.5)  # Simulates slow API call
        response = self.session.get(
            f"{self.base_url}/users/search",
            params={"q": query, "limit": limit}
        )
        response.raise_for_status()
        return response.json()["results"]
```

Our initial test suite covers the basic functionality:

```python
# tests/test_api_client.py
import pytest
from unittest.mock import Mock, patch
from src.api_client import APIClient

@pytest.fixture
def api_client():
    """Create an API client for testing."""
    return APIClient("https://api.example.com", "test-key-123")

@pytest.fixture
def mock_response():
    """Create a mock response object."""
    mock = Mock()
    mock.status_code = 200
    mock.json.return_value = {"id": 1, "username": "testuser"}
    mock.raise_for_status = Mock()
    return mock

def test_get_user_success(api_client, mock_response):
    """Test successful user retrieval."""
    with patch.object(api_client.session, 'get', return_value=mock_response):
        result = api_client.get_user(1)
        assert result["id"] == 1
        assert result["username"] == "testuser"

def test_create_user_success(api_client, mock_response):
    """Test successful user creation."""
    mock_response.json.return_value = {
        "id": 2,
        "username": "newuser",
        "email": "new@example.com"
    }
    with patch.object(api_client.session, 'post', return_value=mock_response):
        result = api_client.create_user("newuser", "new@example.com")
        assert result["username"] == "newuser"

def test_update_user_success(api_client, mock_response):
    """Test successful user update."""
    mock_response.json.return_value = {
        "id": 1,
        "username": "updateduser"
    }
    with patch.object(api_client.session, 'patch', return_value=mock_response):
        result = api_client.update_user(1, {"username": "updateduser"})
        assert result["username"] == "updateduser"

def test_delete_user_success(api_client, mock_response):
    """Test successful user deletion."""
    with patch.object(api_client.session, 'delete', return_value=mock_response):
        result = api_client.delete_user(1)
        assert result is True

def test_search_users_slow(api_client, mock_response):
    """Test user search (slow operation)."""
    mock_response.json.return_value = {
        "results": [
            {"id": 1, "username": "user1"},
            {"id": 2, "username": "user2"}
        ]
    }
    with patch.object(api_client.session, 'get', return_value=mock_response):
        results = api_client.search_users("user")
        assert len(results) == 2
```

Let's run this test suite and observe what happens:

```bash
$ pytest tests/test_api_client.py -v
```

**Output**:
```
============================= test session starts ==============================
platform linux -- Python 3.11.0, pytest-7.4.3, pluggy-1.3.0
cachedir: .pytest_cache
rootdir: /home/user/project
collected 5 items

tests/test_api_client.py::test_get_user_success PASSED                   [ 20%]
tests/test_api_client.py::test_create_user_success PASSED                [ 40%]
tests/test_api_client.py::test_update_user_success PASSED                [ 60%]
tests/test_api_client.py::test_delete_user_success PASSED                [ 80%]
tests/test_api_client.py::test_search_users_slow PASSED                  [100%]

============================== 5 passed in 2.73s ===============================
```

**Current state**: Our tests pass, but we're facing three common problems:

1. **Visibility problem**: The output is functional but not visually appealing or easy to scan
2. **Speed problem**: The test suite takes 2.73 seconds for just 5 tests (the search test includes a 0.5s sleep)
3. **Reporting problem**: We have no persistent record of test results for CI/CD or historical analysis

These are exactly the problems that popular pytest plugins solve. Let's address them one by one.

## pytest-sugar: Making Test Output Beautiful

The first plugin we'll explore is `pytest-sugar`, which transforms pytest's output from functional to delightful.

### Installation and First Run

```bash
$ pip install pytest-sugar
```

Now run the same tests again:

```bash
$ pytest tests/test_api_client.py
```

**Output with pytest-sugar**:
```
 tests/test_api_client.py ✓✓✓✓✓                                        100% ██████████

Results (2.73s):
       5 passed
```

**What changed?**

1. **Visual progress bar**: Instead of line-by-line output, we see a progress bar with checkmarks
2. **Real-time feedback**: Each test completion shows immediately with a ✓ symbol
3. **Cleaner summary**: The final summary is more compact and visually distinct
4. **Color coding**: Passed tests show in green, failures in red (not visible in text)

### Understanding pytest-sugar's Enhancements

pytest-sugar doesn't change how tests run—it changes how results are displayed. Let's see what happens when a test fails:

```python
# tests/test_api_client.py (add this test)
def test_get_user_wrong_id(api_client, mock_response):
    """Test that demonstrates failure output."""
    mock_response.json.return_value = {"id": 999, "username": "wronguser"}
    with patch.object(api_client.session, 'get', return_value=mock_response):
        result = api_client.get_user(1)
        assert result["id"] == 1  # This will fail
```

```bash
$ pytest tests/test_api_client.py::test_get_user_wrong_id
```

**Output**:
```
 tests/test_api_client.py ⨯                                             100% ██████████

―――――――――――――――――――――――――――――――― test_get_user_wrong_id ――――――――――――――――――――――――――――――――

    def test_get_user_wrong_id(api_client, mock_response):
        """Test that demonstrates failure output."""
        mock_response.json.return_value = {"id": 999, "username": "wronguser"}
        with patch.object(api_client.session, 'get', return_value=mock_response):
            result = api_client.get_user(1)
>           assert result["id"] == 1
E           assert 999 == 1

tests/test_api_client.py:58: AssertionError

Results (0.12s):
       1 failed
         - tests/test_api_client.py:53 test_get_user_wrong_id
```

**Key improvements in failure display**:

1. **⨯ symbol**: Immediately shows which test failed in the progress bar
2. **Section separator**: Clear visual boundary around the failure details
3. **Preserved detail**: All pytest's assertion introspection remains intact
4. **Summary with location**: The results section shows exactly where the failure occurred

### When pytest-sugar Shines

**Best for**:
- Development workflows where you're running tests frequently
- Teams that value visual feedback and modern terminal aesthetics
- Large test suites where scanning output quickly matters

**Limitations**:
- Adds minimal overhead (usually imperceptible)
- Some CI systems may not render the enhanced output correctly
- Can be disabled with `pytest --no-sugar` if needed

## pytest-xdist: Parallel Test Execution

Now let's address the speed problem. Our test suite takes 2.73 seconds for 5 tests. As test suites grow to hundreds or thousands of tests, execution time becomes a bottleneck.

### The Problem: Sequential Execution

By default, pytest runs tests sequentially—one after another. Let's add more tests to make the problem more visible:

```python
# tests/test_api_client.py (add these tests)
def test_search_users_multiple_queries(api_client, mock_response):
    """Test multiple search queries."""
    mock_response.json.return_value = {"results": [{"id": 1}]}
    with patch.object(api_client.session, 'get', return_value=mock_response):
        results1 = api_client.search_users("alice")
        results2 = api_client.search_users("bob")
        results3 = api_client.search_users("charlie")
        assert len(results1) == 1

def test_search_users_with_limit(api_client, mock_response):
    """Test search with custom limit."""
    mock_response.json.return_value = {"results": [{"id": i} for i in range(5)]}
    with patch.object(api_client.session, 'get', return_value=mock_response):
        results = api_client.search_users("user", limit=5)
        assert len(results) == 5

def test_search_users_empty_results(api_client, mock_response):
    """Test search with no results."""
    mock_response.json.return_value = {"results": []}
    with patch.object(api_client.session, 'get', return_value=mock_response):
        results = api_client.search_users("nonexistent")
        assert len(results) == 0
```

```bash
$ pytest tests/test_api_client.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 8 items

tests/test_api_client.py::test_get_user_success PASSED                   [ 12%]
tests/test_api_client.py::test_create_user_success PASSED                [ 25%]
tests/test_api_client.py::test_update_user_success PASSED                [ 37%]
tests/test_api_client.py::test_delete_user_success PASSED                [ 50%]
tests/test_api_client.py::test_search_users_slow PASSED                  [ 62%]
tests/test_api_client.py::test_search_users_multiple_queries PASSED      [ 75%]
tests/test_api_client.py::test_search_users_with_limit PASSED            [ 87%]
tests/test_api_client.py::test_search_users_empty_results PASSED         [100%]

============================== 8 passed in 4.23s ===============================
```

**The problem**: 8 tests now take 4.23 seconds. The search tests each include a 0.5s sleep, and they run sequentially. With 4 search tests, that's 2 seconds of just waiting.

### Installing pytest-xdist

```bash
$ pip install pytest-xdist
```

Now run the tests with parallel execution:

```bash
$ pytest tests/test_api_client.py -n auto
```

**Output**:
```
============================= test session starts ==============================
platform linux -- Python 3.11.0, pytest-7.4.3, pluggy-1.3.0
cachedir: .pytest_cache
rootdir: /home/user/project
plugins: xdist-3.5.0
gw0 [8] / gw1 [8] / gw2 [8] / gw3 [8]
........                                                                  [100%]

============================== 8 passed in 1.47s ===============================
```

**What happened?**

1. **Parallel workers**: `gw0`, `gw1`, `gw2`, `gw3` indicate 4 worker processes (auto-detected from CPU cores)
2. **Execution time**: Dropped from 4.23s to 1.47s—a 65% reduction
3. **Test distribution**: pytest-xdist automatically distributed tests across workers

### Understanding the -n Flag

The `-n` flag controls parallelization:

```bash
# Use auto-detection (recommended)
$ pytest -n auto

# Specify exact number of workers
$ pytest -n 4

# Use logical CPU count
$ pytest -n logical
```

### Diagnostic Analysis: How Parallel Execution Works

Let's understand what's happening under the hood by adding some instrumentation:

```python
# tests/test_api_client.py (modify a test)
import os

def test_search_users_slow(api_client, mock_response):
    """Test user search (slow operation)."""
    worker_id = os.environ.get('PYTEST_XDIST_WORKER', 'main')
    print(f"\n[Worker {worker_id}] Running test_search_users_slow")
    
    mock_response.json.return_value = {
        "results": [
            {"id": 1, "username": "user1"},
            {"id": 2, "username": "user2"}
        ]
    }
    with patch.object(api_client.session, 'get', return_value=mock_response):
        results = api_client.search_users("user")
        assert len(results) == 2
```

```bash
$ pytest tests/test_api_client.py::test_search_users_slow -n 2 -s
```

**Output**:
```
============================= test session starts ==============================
gw0 [1] / gw1 [1]

[Worker gw0] Running test_search_users_slow
.                                                                         [100%]

============================== 1 passed in 0.67s ===============================
```

**Key insight**: Each test runs in a separate worker process with its own environment. The `PYTEST_XDIST_WORKER` environment variable identifies which worker is running the test.

### When Parallel Execution Fails: Test Isolation Issues

Parallel execution exposes a critical requirement: **tests must be independent**. Let's see what happens when they're not:

```python
# tests/test_shared_state.py
# WARNING: This demonstrates BAD practice
shared_counter = 0

def test_increment_counter_1():
    """First test that modifies shared state."""
    global shared_counter
    shared_counter += 1
    assert shared_counter == 1

def test_increment_counter_2():
    """Second test that depends on shared state."""
    global shared_counter
    shared_counter += 1
    assert shared_counter == 2

def test_increment_counter_3():
    """Third test that depends on shared state."""
    global shared_counter
    shared_counter += 1
    assert shared_counter == 3
```

```bash
$ pytest tests/test_shared_state.py -v
```

**Output (sequential)**:
```
tests/test_shared_state.py::test_increment_counter_1 PASSED              [ 33%]
tests/test_shared_state.py::test_increment_counter_2 PASSED              [ 66%]
tests/test_shared_state.py::test_increment_counter_3 PASSED              [100%]

============================== 3 passed in 0.01s ===============================
```

Now run with parallelization:

```bash
$ pytest tests/test_shared_state.py -n 2 -v
```

**Output (parallel)**:
```
gw0 [3] / gw1 [3]
tests/test_shared_state.py::test_increment_counter_1 PASSED              [ 33%]
tests/test_shared_state.py::test_increment_counter_2 FAILED              [ 66%]
tests/test_shared_state.py::test_increment_counter_3 PASSED              [100%]

=================================== FAILURES ===================================
__________________________ test_increment_counter_2 ____________________________

    def test_increment_counter_2():
        """Second test that depends on shared state."""
        global shared_counter
        shared_counter += 1
>       assert shared_counter == 2
E       assert 1 == 2

tests/test_shared_state.py:13: AssertionError
```

### Diagnostic Analysis: Reading the Parallel Failure

**The complete output shows**:

1. **Worker distribution**: Tests ran on `gw0` and `gw1` simultaneously
2. **Failure pattern**: `test_increment_counter_2` failed because it ran in a different worker than `test_increment_counter_1`
3. **Root cause**: Each worker has its own process memory—`shared_counter` starts at 0 in each worker

**What this tells us**: Parallel execution reveals hidden dependencies between tests. Tests that pass sequentially but fail in parallel are **not truly independent**.

**The fix**: Use fixtures for setup instead of shared state:

```python
# tests/test_shared_state_fixed.py
import pytest

@pytest.fixture
def counter():
    """Provide a fresh counter for each test."""
    return {"value": 0}

def test_increment_counter_1(counter):
    """First test with isolated state."""
    counter["value"] += 1
    assert counter["value"] == 1

def test_increment_counter_2(counter):
    """Second test with isolated state."""
    counter["value"] += 1
    assert counter["value"] == 1  # Each test starts fresh

def test_increment_counter_3(counter):
    """Third test with isolated state."""
    counter["value"] += 1
    assert counter["value"] == 1
```

```bash
$ pytest tests/test_shared_state_fixed.py -n 2 -v
```

**Output**:
```
gw0 [3] / gw1 [3]
tests/test_shared_state_fixed.py::test_increment_counter_1 PASSED        [ 33%]
tests/test_shared_state_fixed.py::test_increment_counter_2 PASSED        [ 66%]
tests/test_shared_state_fixed.py::test_increment_counter_3 PASSED        [100%]

============================== 3 passed in 0.23s ===============================
```

**Result**: All tests pass in parallel because each has isolated state through fixtures.

### When to Use pytest-xdist

**Best for**:
- Large test suites (100+ tests) where execution time matters
- CI/CD pipelines where faster feedback is critical
- Tests that are truly independent (no shared state, no database dependencies)

**Avoid when**:
- Tests share resources (databases, files, network ports)
- Tests have strict ordering requirements
- Debugging failures (parallel output is harder to read)

**Pro tip**: Use `-n 0` to disable parallelization temporarily:

```bash
$ pytest -n 0  # Forces sequential execution even if xdist is installed
```

## pytest-html: Generating Test Reports

The third common need is **persistent test reports**. Console output disappears after the test run, but HTML reports provide:

1. **Historical record**: Archive test results over time
2. **CI/CD integration**: Attach reports to build artifacts
3. **Stakeholder communication**: Share results with non-technical team members

### Installation and Basic Usage

```bash
$ pip install pytest-html
```

Generate a report for our API client tests:

```bash
$ pytest tests/test_api_client.py --html=report.html --self-contained-html
```

**Output**:
```
============================= test session starts ==============================
collected 8 items

tests/test_api_client.py ........                                        [100%]

============================== 8 passed in 2.15s ===============================
Generated html report: file:///home/user/project/report.html
```

**What was created?**

The `--self-contained-html` flag creates a single HTML file with all assets embedded. Open `report.html` in a browser to see:

1. **Summary section**: Total tests, pass/fail counts, duration
2. **Environment details**: Python version, pytest version, plugins
3. **Test results table**: Each test with status, duration, and details
4. **Expandable failures**: Click to see full traceback and assertion details

### Customizing Report Content

Let's add more information to our tests that will appear in the report:

```python
# tests/test_api_client.py (modify tests)
import pytest

def test_get_user_success(api_client, mock_response):
    """Test successful user retrieval."""
    # Add extra information for the report
    pytest.extra = {
        "endpoint": "/users/1",
        "method": "GET",
        "expected_status": 200
    }
    
    with patch.object(api_client.session, 'get', return_value=mock_response):
        result = api_client.get_user(1)
        assert result["id"] == 1
        assert result["username"] == "testuser"
```

### Adding Screenshots and Logs to Reports

For web testing or complex scenarios, you can attach additional artifacts:

```python
# tests/conftest.py
import pytest
from datetime import datetime

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Hook to add extra information to test reports."""
    outcome = yield
    report = outcome.get_result()
    
    # Add timestamp to all tests
    report.timestamp = datetime.now().isoformat()
    
    # Add extra details for failures
    if report.when == "call" and report.failed:
        # You could add screenshots, logs, etc. here
        report.extra_info = {
            "test_name": item.name,
            "failure_time": datetime.now().isoformat()
        }
```

### Combining Plugins: The Complete Workflow

Now let's use all three plugins together for the optimal development experience:

```bash
$ pytest tests/test_api_client.py -n auto --html=report.html --self-contained-html
```

**Output**:
```
 tests/test_api_client.py ✓✓✓✓✓✓✓✓                                     100% ██████████
gw0 [8] / gw1 [8] / gw2 [8] / gw3 [8]

Results (1.52s):
       8 passed

Generated html report: file:///home/user/project/report.html
```

**What we achieved**:

1. **pytest-sugar**: Beautiful, real-time visual feedback during test execution
2. **pytest-xdist**: Parallel execution reduced time from 4.23s to 1.52s (64% faster)
3. **pytest-html**: Persistent report for documentation and CI/CD integration

### Configuration for Consistent Usage

Instead of typing flags every time, configure plugins in `pytest.ini`:

```ini
# pytest.ini
[pytest]
addopts = 
    -n auto
    --html=reports/test_report.html
    --self-contained-html
    -v

# Disable plugins when needed
# addopts = --no-sugar -n 0
```

Now simply run:

```bash
$ pytest tests/test_api_client.py
```

All configured options apply automatically.

## Decision Framework: Which Plugins When?

| Plugin | Primary Benefit | Primary Cost | Best For | Avoid When |
|--------|----------------|--------------|----------|------------|
| **pytest-sugar** | Visual clarity | Minimal overhead | Development, frequent test runs | CI systems with poor terminal support |
| **pytest-xdist** | Speed (parallelization) | Requires test isolation | Large suites, CI/CD | Tests with shared state, debugging |
| **pytest-html** | Persistent reports | Disk space, slight overhead | CI/CD, documentation | Local development (unless needed) |

### Combining Plugins: Compatibility Matrix

| Combination | Compatible? | Notes |
|-------------|-------------|-------|
| sugar + xdist | ✅ Yes | Sugar shows parallel progress |
| sugar + html | ✅ Yes | Sugar affects console, html affects file |
| xdist + html | ✅ Yes | Report includes parallel execution details |
| All three | ✅ Yes | Recommended for CI/CD pipelines |

## The Journey: From Basic Tests to Optimized Workflow

| Stage | Problem | Plugin Solution | Result |
|-------|---------|----------------|--------|
| 0 | Basic tests work but output is plain | None | Functional but not optimal |
| 1 | Hard to scan test results quickly | pytest-sugar | Visual feedback, easier scanning |
| 2 | Test suite takes too long | pytest-xdist | 64% faster execution |
| 3 | No persistent record of results | pytest-html | Archived reports for CI/CD |
| 4 | Want all benefits together | All three + config | Optimal development workflow |

### Lessons Learned

1. **Plugins solve real problems**: Each plugin addresses a specific pain point in the testing workflow
2. **Composition over configuration**: Plugins work together without conflicts
3. **Configuration matters**: Use `pytest.ini` to make plugin usage consistent across the team
4. **Test isolation is critical**: Parallel execution exposes hidden dependencies between tests
5. **Choose based on context**: Development workflows differ from CI/CD requirements

## Installing and Configuring Plugins

## Installing and Configuring Plugins

Now that we've seen what plugins can do, let's understand how to install, configure, and manage them systematically. Plugin management becomes critical as your test suite grows and your team adopts more tools.

### The Plugin Discovery Mechanism

Before we install plugins, let's understand how pytest finds and loads them. This knowledge will help you troubleshoot plugin issues and understand configuration options.

#### How Pytest Discovers Plugins

Pytest uses Python's entry point system to discover plugins automatically. When you install a plugin with pip, it registers itself with pytest.

Let's see which plugins are currently active:

```bash
$ pytest --version
```

**Output**:
```
pytest 7.4.3
plugins: sugar-0.9.7, xdist-3.5.0, html-4.1.1
```

This shows pytest found three plugins. Let's get more detail:

```bash
$ pytest --trace-config
```

**Output**:
```
============================= test session starts ==============================
active plugins:
    sugar-0.9.7: /path/to/venv/lib/python3.11/site-packages/pytest_sugar.py
    xdist-3.5.0: /path/to/venv/lib/python3.11/site-packages/xdist/plugin.py
    html-4.1.1: /path/to/venv/lib/python3.11/site-packages/pytest_html/plugin.py
```

**What this tells us**:

1. **Plugin names**: The registered name (e.g., `sugar-0.9.7`)
2. **Plugin location**: Where the plugin code lives
3. **Load order**: Plugins load in the order shown

### Installation Methods

#### Method 1: Direct Installation (Development)

For local development, install plugins directly:

```bash
# Install a single plugin
$ pip install pytest-cov

# Install multiple plugins
$ pip install pytest-cov pytest-timeout pytest-mock

# Install with specific version
$ pip install pytest-cov==4.1.0
```

#### Method 2: Requirements File (Team Projects)

For team projects, maintain a `requirements-dev.txt`:

```text
# requirements-dev.txt
pytest==7.4.3
pytest-cov==4.1.0
pytest-xdist==3.5.0
pytest-html==4.1.1
pytest-sugar==0.9.7
pytest-timeout==2.2.0
pytest-mock==3.12.0
```

```bash
$ pip install -r requirements-dev.txt
```

#### Method 3: pyproject.toml (Modern Projects)

For modern Python projects using `pyproject.toml`:

```toml
# pyproject.toml
[project.optional-dependencies]
dev = [
    "pytest>=7.4.3",
    "pytest-cov>=4.1.0",
    "pytest-xdist>=3.5.0",
    "pytest-html>=4.1.1",
    "pytest-sugar>=0.9.7",
    "pytest-timeout>=2.2.0",
    "pytest-mock>=3.12.0",
]
```

```bash
$ pip install -e ".[dev]"
```

### Configuration Hierarchy

Pytest plugins can be configured in multiple places. Understanding the hierarchy helps you control plugin behavior effectively.

#### Configuration Priority (Highest to Lowest)

1. **Command-line flags**: Override everything
2. **Environment variables**: Override config files
3. **pytest.ini / pyproject.toml**: Project-level defaults
4. **conftest.py**: Programmatic configuration
5. **Plugin defaults**: Built-in plugin settings

Let's demonstrate this hierarchy with a concrete example using pytest-timeout:

```bash
$ pip install pytest-timeout
```

#### Level 1: Plugin Defaults

Without any configuration, pytest-timeout uses its defaults (no timeout):

```python
# tests/test_timeout.py
import time

def test_slow_operation():
    """Test that takes 2 seconds."""
    time.sleep(2)
    assert True
```

```bash
$ pytest tests/test_timeout.py -v
```

**Output**:
```
tests/test_timeout.py::test_slow_operation PASSED                        [100%]

============================== 1 passed in 2.01s ===============================
```

Test passes because no timeout is configured.

#### Level 2: Configuration File

Add timeout to `pytest.ini`:

```ini
# pytest.ini
[pytest]
timeout = 1
```

```bash
$ pytest tests/test_timeout.py -v
```

**Output**:
```
tests/test_timeout.py::test_slow_operation FAILED                        [100%]

=================================== FAILURES ===================================
__________________________ test_slow_operation _________________________________
E   Failed: Timeout >1.0s

tests/test_timeout.py:5: Failed
============================== 1 failed in 1.23s ===============================
```

**Diagnostic Analysis**:

1. **Summary line**: `FAILED` with `Timeout >1.0s` message
2. **Root cause**: Test exceeded the 1-second timeout configured in pytest.ini
3. **What this tells us**: Configuration file settings apply to all tests

#### Level 3: Environment Variables

Override the config file with an environment variable:

```bash
$ PYTEST_TIMEOUT=3 pytest tests/test_timeout.py -v
```

**Output**:
```
tests/test_timeout.py::test_slow_operation PASSED                        [100%]

============================== 1 passed in 2.01s ===============================
```

Test passes because environment variable (3 seconds) overrides pytest.ini (1 second).

#### Level 4: Command-Line Flags

Override everything with a command-line flag:

```bash
$ PYTEST_TIMEOUT=3 pytest tests/test_timeout.py -v --timeout=0.5
```

**Output**:
```
tests/test_timeout.py::test_slow_operation FAILED                        [100%]

=================================== FAILURES ===================================
__________________________ test_slow_operation _________________________________
E   Failed: Timeout >0.5s

tests/test_timeout.py:5: Failed
============================== 1 failed in 0.73s ===============================
```

Command-line flag (0.5 seconds) overrides both environment variable and config file.

### Common Configuration Patterns

#### Pattern 1: Different Settings for Different Environments

Use environment variables to adjust plugin behavior:

```bash
# Development: No timeout, pretty output
$ pytest

# CI: Strict timeout, parallel execution, HTML report
$ PYTEST_TIMEOUT=30 pytest -n auto --html=report.html
```

#### Pattern 2: Per-Test Configuration

Some plugins support per-test configuration via markers:

```python
# tests/test_timeout.py
import pytest
import time

@pytest.mark.timeout(5)
def test_slow_operation_allowed():
    """This test gets 5 seconds instead of the default."""
    time.sleep(3)
    assert True

@pytest.mark.timeout(0)  # Disable timeout for this test
def test_very_slow_operation():
    """This test has no timeout."""
    time.sleep(10)
    assert True
```

#### Pattern 3: Conditional Plugin Activation

Disable plugins conditionally:

```ini
# pytest.ini
[pytest]
# Enable plugins by default
addopts = -n auto --html=report.html

# Override in specific scenarios
# pytest -p no:xdist  # Disable xdist
# pytest -p no:sugar  # Disable sugar
```

### Plugin Configuration Reference

Let's configure the plugins we've used so far comprehensively:

#### pytest-xdist Configuration

```ini
# pytest.ini
[pytest]
# Number of workers (auto, logical, or specific number)
addopts = -n auto

# Distribution strategy
# - load: distribute by file (default)
# - loadscope: distribute by test scope (class, module)
# - loadfile: distribute by file, but keep file tests together
# - no: disable distribution
# Example: --dist=loadscope
```

**Advanced xdist options**:

```bash
# Run tests on remote machines
$ pytest -n 4 --tx ssh=user@host1 --tx ssh=user@host2

# Control test distribution
$ pytest --dist=loadscope  # Keep test classes together

# Maximum workers
$ pytest -n 8 --maxprocesses=4  # Use max 4 processes even if more CPUs
```

#### pytest-html Configuration

```ini
# pytest.ini
[pytest]
# Basic HTML report
addopts = --html=reports/report.html --self-contained-html

# Additional options
# --html=report.html              # Report location
# --self-contained-html           # Embed assets
# --css=custom.css                # Custom styling
# --html-report=report.html       # Alternative syntax
```

**Customizing HTML reports**:

```python
# conftest.py
import pytest
from datetime import datetime

@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Add custom information to HTML report."""
    outcome = yield
    report = outcome.get_result()
    
    # Add custom metadata
    if report.when == "call":
        report.user_properties.append(("timestamp", datetime.now().isoformat()))
        report.user_properties.append(("test_id", item.nodeid))

def pytest_html_report_title(report):
    """Customize report title."""
    report.title = "API Client Test Report"

def pytest_configure(config):
    """Add custom metadata to report."""
    config._metadata["Project"] = "API Client"
    config._metadata["Tester"] = "QA Team"
```

#### pytest-sugar Configuration

pytest-sugar has minimal configuration but can be disabled:

```bash
# Disable sugar temporarily
$ pytest --no-sugar

# Disable via plugin system
$ pytest -p no:sugar
```

### Managing Plugin Conflicts

Sometimes plugins conflict with each other or with your test code. Let's see how to diagnose and resolve conflicts.

#### Conflict Example: Output Capture

Let's create a scenario where plugins interfere with each other:

```python
# tests/test_output.py
def test_with_print():
    """Test that prints output."""
    print("Debug information")
    print("More debug information")
    assert True
```

```bash
$ pytest tests/test_output.py -v -s
```

**Output with pytest-sugar**:
```
 tests/test_output.py ✓                                                 100% ██████████
Debug information
More debug information

Results (0.01s):
       1 passed
```

**Output without pytest-sugar**:

```bash
$ pytest tests/test_output.py -v -s -p no:sugar
```

**Output**:
```
tests/test_output.py::test_with_print Debug information
More debug information
PASSED

============================== 1 passed in 0.01s ===============================
```

**Diagnostic Analysis**:

1. **With sugar**: Output appears after the progress bar
2. **Without sugar**: Output appears inline with test execution
3. **Root cause**: pytest-sugar buffers output for cleaner display
4. **Solution**: Use `-p no:sugar` when you need immediate output visibility

#### Conflict Resolution Strategies

**Strategy 1: Selective Plugin Disabling**

```bash
# Disable specific plugin for one run
$ pytest -p no:sugar

# Disable multiple plugins
$ pytest -p no:sugar -p no:xdist
```

**Strategy 2: Plugin Load Order Control**

Some plugins must load before others. Control this via `conftest.py`:

```python
# conftest.py
pytest_plugins = [
    "pytest_html",  # Load first
    "pytest_sugar", # Load second
]
```

**Strategy 3: Configuration Isolation**

Use different config files for different scenarios:

```ini
# pytest-dev.ini (for development)
[pytest]
addopts = -v --no-sugar

# pytest-ci.ini (for CI/CD)
[pytest]
addopts = -n auto --html=report.html --self-contained-html
```

```bash
# Use specific config
$ pytest -c pytest-dev.ini
$ pytest -c pytest-ci.ini
```

### Verifying Plugin Configuration

After configuring plugins, verify they're working as expected:

#### Verification Checklist

**1. Check plugin loading**:

```bash
$ pytest --version
# Should list all installed plugins

$ pytest --trace-config
# Shows detailed plugin information
```

**2. Test plugin functionality**:

```bash
# Test xdist
$ pytest --collect-only -n 2
# Should show worker distribution

# Test html
$ pytest --collect-only --html=test.html
# Should create HTML file

# Test sugar
$ pytest tests/ -v
# Should show progress bar
```

**3. Verify configuration application**:

```bash
# Show effective configuration
$ pytest --showlocals --tb=short --help
# Lists all active options including plugin options
```

### Complete Configuration Example

Here's a production-ready configuration combining all best practices:

```toml
# pyproject.toml
[tool.pytest.ini_options]
minversion = "7.0"
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]

# Plugin configuration
addopts = """
    -v
    --strict-markers
    --tb=short
    --cov=src
    --cov-report=html
    --cov-report=term-missing
    --html=reports/test_report.html
    --self-contained-html
"""

# Markers
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]

# Timeout configuration
timeout = 300
timeout_method = "thread"

# Coverage configuration
[tool.coverage.run]
source = ["src"]
omit = ["*/tests/*", "*/test_*.py"]

[tool.coverage.report]
exclude_lines = [
    "pragma: no cover",
    "def __repr__",
    "raise AssertionError",
    "raise NotImplementedError",
    "if __name__ == .__main__.:",
]
```

**Environment-specific overrides**:

```bash
# Development: Fast feedback, no coverage
$ pytest -n auto -p no:cov

# CI: Full coverage, parallel, reports
$ pytest -n auto --cov --html=report.html

# Debugging: Sequential, verbose, no sugar
$ pytest -n 0 -vv -p no:sugar --pdb
```

## Decision Framework: Configuration Strategies

| Scenario | Configuration Method | Rationale |
|----------|---------------------|-----------|
| **Local development** | pytest.ini with minimal options | Fast feedback, easy to override |
| **Team project** | pyproject.toml with comprehensive options | Single source of truth, version controlled |
| **CI/CD pipeline** | Command-line flags + environment variables | Flexibility, environment-specific settings |
| **Multiple environments** | Separate config files (pytest-dev.ini, pytest-ci.ini) | Clear separation, explicit control |
| **Plugin conflicts** | Selective disabling with -p no:plugin | Temporary fixes, debugging |

### Lessons Learned

1. **Configuration hierarchy matters**: Understand which settings override others
2. **Start minimal, add as needed**: Don't configure everything upfront
3. **Document your choices**: Explain why specific plugins are configured certain ways
4. **Test your configuration**: Verify plugins work as expected after configuration changes
5. **Environment-specific settings**: Use different configurations for development vs. CI/CD
6. **Plugin conflicts are solvable**: Use selective disabling and load order control

## Creating Custom Plugins

## Creating Custom Plugins

Now that we understand how to use and configure existing plugins, let's learn how to create our own. Custom plugins let you extend pytest with project-specific functionality that doesn't exist in the ecosystem.

### When to Create a Custom Plugin

Before writing a plugin, ask yourself:

1. **Is this functionality reusable across multiple test files?** If yes, consider a plugin.
2. **Does this solve a problem specific to your project?** If yes, a local plugin might be appropriate.
3. **Could this benefit the wider community?** If yes, consider publishing it.

Let's build a custom plugin that solves a real problem: **tracking test execution time and automatically marking slow tests**.

### Phase 1: The Problem - Identifying Slow Tests

Our API client test suite has grown, and some tests are noticeably slower than others. We want to:

1. Automatically identify tests that exceed a time threshold
2. Mark them with a custom marker for easy filtering
3. Generate a report of slow tests

Let's start with our current test suite:

```python
# tests/test_api_performance.py
import time
import pytest
from unittest.mock import Mock, patch
from src.api_client import APIClient

@pytest.fixture
def api_client():
    return APIClient("https://api.example.com", "test-key")

@pytest.fixture
def mock_response():
    mock = Mock()
    mock.status_code = 200
    mock.json.return_value = {"id": 1, "data": "test"}
    mock.raise_for_status = Mock()
    return mock

def test_fast_operation(api_client, mock_response):
    """Fast test - completes in <0.1s."""
    with patch.object(api_client.session, 'get', return_value=mock_response):
        result = api_client.get_user(1)
        assert result["id"] == 1

def test_medium_operation(api_client, mock_response):
    """Medium test - completes in ~0.3s."""
    time.sleep(0.3)
    with patch.object(api_client.session, 'get', return_value=mock_response):
        result = api_client.get_user(1)
        assert result["id"] == 1

def test_slow_operation(api_client, mock_response):
    """Slow test - completes in ~1s."""
    time.sleep(1.0)
    with patch.object(api_client.session, 'get', return_value=mock_response):
        result = api_client.get_user(1)
        assert result["id"] == 1

def test_very_slow_operation(api_client, mock_response):
    """Very slow test - completes in ~2s."""
    time.sleep(2.0)
    with patch.object(api_client.session, 'get', return_value=mock_response):
        result = api_client.get_user(1)
        assert result["id"] == 1
```

```bash
$ pytest tests/test_api_performance.py -v --durations=0
```

**Output**:
```
tests/test_api_performance.py::test_fast_operation PASSED               [ 25%]
tests/test_api_performance.py::test_medium_operation PASSED             [ 50%]
tests/test_api_performance.py::test_slow_operation PASSED               [ 75%]
tests/test_api_performance.py::test_very_slow_operation PASSED          [100%]

============================== slowest durations ===============================
2.00s call     tests/test_api_performance.py::test_very_slow_operation
1.00s call     tests/test_api_performance.py::test_slow_operation
0.30s call     tests/test_api_performance.py::test_medium_operation
0.01s call     tests/test_api_performance.py::test_fast_operation

============================== 4 passed in 3.32s ===============================
```

**Current limitation**: We can see slow tests with `--durations`, but:

1. We have to remember to use the flag
2. We can't filter slow tests with `-m`
3. We have no automated way to track slow tests over time
4. The threshold is not configurable

**What we need**: A plugin that automatically marks tests exceeding a threshold and provides filtering capabilities.

### Phase 2: Plugin Structure - The Basics

A pytest plugin is simply a Python module that defines hook functions. Let's create our first plugin:

```python
# conftest.py (local plugin)
import pytest

def pytest_configure(config):
    """Register custom markers."""
    config.addinivalue_line(
        "markers",
        "slow: marks tests as slow (automatically applied based on duration)"
    )

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Hook that runs after each test phase."""
    outcome = yield
    report = outcome.get_result()
    
    # Only process the actual test call (not setup/teardown)
    if report.when == "call":
        print(f"\n[DEBUG] Test {item.name} took {report.duration:.2f}s")
```

```bash
$ pytest tests/test_api_performance.py::test_fast_operation -v -s
```

**Output**:
```
tests/test_api_performance.py::test_fast_operation 
[DEBUG] Test test_fast_operation took 0.01s
PASSED

============================== 1 passed in 0.02s ===============================
```

**What happened?**

1. **pytest_configure**: Registered our custom `slow` marker
2. **pytest_runtest_makereport**: Hook that runs after each test phase
3. **hookwrapper=True**: Allows us to wrap around the normal hook execution
4. **report.when == "call"**: Filters to only the actual test execution (not setup/teardown)

### Diagnostic Analysis: Understanding Hook Execution

Let's add more instrumentation to understand the hook lifecycle:

```python
# conftest.py
import pytest

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Hook that runs after each test phase."""
    print(f"\n[HOOK] pytest_runtest_makereport called")
    print(f"  item.name: {item.name}")
    print(f"  call.when: {call.when}")
    
    outcome = yield
    report = outcome.get_result()
    
    print(f"  report.when: {report.when}")
    print(f"  report.duration: {report.duration:.4f}s")
    print(f"  report.outcome: {report.outcome}")
```

```bash
$ pytest tests/test_api_performance.py::test_fast_operation -v -s
```

**Output**:
```
[HOOK] pytest_runtest_makereport called
  item.name: test_fast_operation
  call.when: setup
  report.when: setup
  report.duration: 0.0001s
  report.outcome: passed

[HOOK] pytest_runtest_makereport called
  item.name: test_fast_operation
  call.when: call
  report.when: call
  report.duration: 0.0089s
  report.outcome: passed

[HOOK] pytest_runtest_makereport called
  item.name: test_fast_operation
  call.when: teardown
  report.when: teardown
  report.duration: 0.0001s
  report.outcome: passed

PASSED

============================== 1 passed in 0.01s ===============================
```

**Key insights**:

1. **Three phases**: Hook runs three times per test (setup, call, teardown)
2. **Duration tracking**: Each phase has its own duration
3. **Outcome tracking**: Each phase can pass or fail independently
4. **call phase**: This is where the actual test runs—what we want to measure

### Phase 3: Implementing Automatic Slow Test Marking

Now let's implement the core functionality: automatically mark tests that exceed a threshold:

```python
# conftest.py
import pytest

# Configuration
SLOW_TEST_THRESHOLD = 0.5  # seconds

def pytest_configure(config):
    """Register custom markers."""
    config.addinivalue_line(
        "markers",
        "slow: marks tests as slow (automatically applied based on duration)"
    )

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Automatically mark slow tests."""
    outcome = yield
    report = outcome.get_result()
    
    # Only process the actual test call
    if report.when == "call" and report.duration > SLOW_TEST_THRESHOLD:
        # Add the slow marker to the test item
        item.add_marker(pytest.mark.slow)
        
        # Store duration for later reporting
        if not hasattr(item, "slow_duration"):
            item.slow_duration = report.duration
```

```bash
$ pytest tests/test_api_performance.py -v -m slow
```

**Output**:
```
tests/test_api_performance.py::test_slow_operation PASSED               [ 50%]
tests/test_api_performance.py::test_very_slow_operation PASSED          [100%]

============================== 2 passed in 3.01s ===============================
```

**What we achieved**:

1. **Automatic marking**: Tests exceeding 0.5s were automatically marked as `slow`
2. **Filtering works**: `-m slow` successfully filtered to only slow tests
3. **No manual marking needed**: Developers don't need to remember to add `@pytest.mark.slow`

### Phase 4: Making Configuration Flexible

Hardcoding the threshold isn't ideal. Let's make it configurable:

```python
# conftest.py
import pytest

def pytest_addoption(parser):
    """Add custom command-line options."""
    parser.addoption(
        "--slow-threshold",
        action="store",
        default=0.5,
        type=float,
        help="Threshold in seconds for marking tests as slow (default: 0.5)"
    )

def pytest_configure(config):
    """Register custom markers and store configuration."""
    config.addinivalue_line(
        "markers",
        "slow: marks tests as slow (automatically applied based on duration)"
    )
    
    # Store threshold for use in hooks
    config.slow_threshold = config.getoption("--slow-threshold")

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Automatically mark slow tests."""
    outcome = yield
    report = outcome.get_result()
    
    # Get threshold from config
    threshold = item.config.slow_threshold
    
    if report.when == "call" and report.duration > threshold:
        item.add_marker(pytest.mark.slow)
        item.slow_duration = report.duration
```

Now we can configure the threshold:

```bash
# Use default threshold (0.5s)
$ pytest tests/test_api_performance.py -v -m slow

# Use custom threshold (1.0s)
$ pytest tests/test_api_performance.py -v -m slow --slow-threshold=1.0
```

**Output with --slow-threshold=1.0**:
```
tests/test_api_performance.py::test_very_slow_operation PASSED          [100%]

============================== 1 passed in 2.01s ===============================
```

Only the 2-second test is marked as slow now.

### Phase 5: Adding a Summary Report

Let's add a summary report showing all slow tests at the end:

```python
# conftest.py
import pytest

def pytest_addoption(parser):
    """Add custom command-line options."""
    parser.addoption(
        "--slow-threshold",
        action="store",
        default=0.5,
        type=float,
        help="Threshold in seconds for marking tests as slow"
    )
    parser.addoption(
        "--show-slow-summary",
        action="store_true",
        default=False,
        help="Show summary of slow tests at the end"
    )

def pytest_configure(config):
    """Register custom markers and initialize storage."""
    config.addinivalue_line(
        "markers",
        "slow: marks tests as slow (automatically applied based on duration)"
    )
    config.slow_threshold = config.getoption("--slow-threshold")
    config.slow_tests = []  # Store slow test information

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Automatically mark slow tests and collect information."""
    outcome = yield
    report = outcome.get_result()
    
    threshold = item.config.slow_threshold
    
    if report.when == "call" and report.duration > threshold:
        item.add_marker(pytest.mark.slow)
        
        # Store slow test information
        item.config.slow_tests.append({
            "nodeid": item.nodeid,
            "duration": report.duration,
            "threshold": threshold
        })

def pytest_terminal_summary(terminalreporter, exitstatus, config):
    """Add custom section to terminal summary."""
    if not config.getoption("--show-slow-summary"):
        return
    
    slow_tests = config.slow_tests
    if not slow_tests:
        return
    
    # Sort by duration (slowest first)
    slow_tests.sort(key=lambda x: x["duration"], reverse=True)
    
    terminalreporter.section("Slow Tests Summary")
    terminalreporter.write_line(
        f"Found {len(slow_tests)} test(s) exceeding {config.slow_threshold}s threshold:\n"
    )
    
    for test in slow_tests:
        terminalreporter.write_line(
            f"  {test['duration']:.2f}s - {test['nodeid']}"
        )
```

```bash
$ pytest tests/test_api_performance.py -v --show-slow-summary
```

**Output**:
```
tests/test_api_performance.py::test_fast_operation PASSED               [ 25%]
tests/test_api_performance.py::test_medium_operation PASSED             [ 50%]
tests/test_api_performance.py::test_slow_operation PASSED               [ 75%]
tests/test_api_performance.py::test_very_slow_operation PASSED          [100%]

============================== Slow Tests Summary ==============================
Found 2 test(s) exceeding 0.5s threshold:

  2.00s - tests/test_api_performance.py::test_very_slow_operation
  1.00s - tests/test_api_performance.py::test_slow_operation

============================== 4 passed in 3.32s ===============================
```

**What we achieved**:

1. **Custom summary section**: Added a new section to pytest's terminal output
2. **Sorted by duration**: Slowest tests appear first
3. **Configurable display**: Only shows when `--show-slow-summary` is used
4. **Actionable information**: Developers can immediately see which tests to optimize

### Phase 6: Adding Configuration File Support

Let's make the plugin configurable via `pytest.ini`:

```ini
# pytest.ini
[pytest]
slow_threshold = 0.5
show_slow_summary = true
```

```python
# conftest.py (updated)
import pytest

def pytest_addoption(parser):
    """Add custom command-line options."""
    parser.addini(
        "slow_threshold",
        type="string",
        default="0.5",
        help="Threshold in seconds for marking tests as slow"
    )
    parser.addini(
        "show_slow_summary",
        type="bool",
        default=False,
        help="Show summary of slow tests at the end"
    )
    
    # Also support command-line override
    parser.addoption(
        "--slow-threshold",
        action="store",
        type=float,
        help="Override slow_threshold from config"
    )

def pytest_configure(config):
    """Register custom markers and initialize storage."""
    config.addinivalue_line(
        "markers",
        "slow: marks tests as slow (automatically applied based on duration)"
    )
    
    # Get threshold from command-line or config file
    threshold = config.getoption("--slow-threshold")
    if threshold is None:
        threshold = float(config.getini("slow_threshold"))
    
    config.slow_threshold = threshold
    config.slow_tests = []

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Automatically mark slow tests and collect information."""
    outcome = yield
    report = outcome.get_result()
    
    threshold = item.config.slow_threshold
    
    if report.when == "call" and report.duration > threshold:
        item.add_marker(pytest.mark.slow)
        item.config.slow_tests.append({
            "nodeid": item.nodeid,
            "duration": report.duration,
            "threshold": threshold
        })

def pytest_terminal_summary(terminalreporter, exitstatus, config):
    """Add custom section to terminal summary."""
    show_summary = config.getini("show_slow_summary")
    
    if not show_summary:
        return
    
    slow_tests = config.slow_tests
    if not slow_tests:
        return
    
    slow_tests.sort(key=lambda x: x["duration"], reverse=True)
    
    terminalreporter.section("Slow Tests Summary")
    terminalreporter.write_line(
        f"Found {len(slow_tests)} test(s) exceeding {config.slow_threshold}s threshold:\n"
    )
    
    for test in slow_tests:
        terminalreporter.write_line(
            f"  {test['duration']:.2f}s - {test['nodeid']}"
        )
```

Now the plugin reads configuration from `pytest.ini` by default:

```bash
# Uses pytest.ini configuration
$ pytest tests/test_api_performance.py -v

# Override with command-line
$ pytest tests/test_api_performance.py -v --slow-threshold=1.0
```

### The Complete Plugin: Production-Ready Version

Here's the final, production-ready version with error handling and documentation:

```python
# conftest.py
"""
Pytest plugin for automatic slow test detection and reporting.

This plugin automatically marks tests that exceed a configurable duration
threshold and provides filtering and reporting capabilities.

Configuration:
    pytest.ini:
        slow_threshold = 0.5  # seconds
        show_slow_summary = true

Command-line:
    --slow-threshold=1.0      # Override threshold
    -m slow                   # Run only slow tests
    -m "not slow"             # Skip slow tests

Example:
    $ pytest -v --show-slow-summary
    $ pytest -m slow
    $ pytest --slow-threshold=1.0
"""
import pytest
from typing import Dict, List, Any

def pytest_addoption(parser):
    """Add custom command-line options and ini-file values."""
    parser.addini(
        "slow_threshold",
        type="string",
        default="0.5",
        help="Threshold in seconds for marking tests as slow (default: 0.5)"
    )
    parser.addini(
        "show_slow_summary",
        type="bool",
        default=False,
        help="Show summary of slow tests at the end (default: False)"
    )
    
    parser.addoption(
        "--slow-threshold",
        action="store",
        type=float,
        help="Override slow_threshold from config file"
    )

def pytest_configure(config):
    """Register custom markers and initialize plugin state."""
    config.addinivalue_line(
        "markers",
        "slow: marks tests as slow (automatically applied based on duration)"
    )
    
    # Get threshold from command-line or config file
    threshold = config.getoption("--slow-threshold")
    if threshold is None:
        try:
            threshold = float(config.getini("slow_threshold"))
        except (ValueError, TypeError):
            threshold = 0.5  # Fallback to default
    
    # Validate threshold
    if threshold < 0:
        raise pytest.UsageError(
            f"slow_threshold must be non-negative, got {threshold}"
        )
    
    config.slow_threshold = threshold
    config.slow_tests: List[Dict[str, Any]] = []

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """
    Automatically mark slow tests and collect information.
    
    This hook runs after each test phase (setup, call, teardown).
    We only process the 'call' phase to measure actual test execution time.
    """
    outcome = yield
    report = outcome.get_result()
    
    # Only process the actual test call (not setup/teardown)
    if report.when != "call":
        return
    
    threshold = item.config.slow_threshold
    
    # Mark test as slow if it exceeds threshold
    if report.duration > threshold:
        # Add marker for filtering
        item.add_marker(pytest.mark.slow)
        
        # Store information for summary report
        item.config.slow_tests.append({
            "nodeid": item.nodeid,
            "duration": report.duration,
            "threshold": threshold,
            "outcome": report.outcome
        })

def pytest_terminal_summary(terminalreporter, exitstatus, config):
    """
    Add custom section to terminal summary showing slow tests.
    
    This hook runs at the end of the test session to display
    a summary of all tests that exceeded the slow threshold.
    """
    # Check if summary should be shown
    show_summary = config.getini("show_slow_summary")
    if not show_summary:
        return
    
    slow_tests = config.slow_tests
    if not slow_tests:
        return
    
    # Sort by duration (slowest first)
    slow_tests.sort(key=lambda x: x["duration"], reverse=True)
    
    # Add custom section to output
    terminalreporter.section("Slow Tests Summary")
    terminalreporter.write_line(
        f"Found {len(slow_tests)} test(s) exceeding "
        f"{config.slow_threshold}s threshold:\n"
    )
    
    # Display each slow test with duration and status
    for test in slow_tests:
        status_symbol = "✓" if test["outcome"] == "passed" else "✗"
        terminalreporter.write_line(
            f"  {status_symbol} {test['duration']:.2f}s - {test['nodeid']}"
        )
    
    # Add helpful tip
    terminalreporter.write_line(
        f"\nTip: Run 'pytest -m slow' to run only slow tests"
    )
```

### Testing the Plugin

Let's verify our plugin works correctly:

```bash
# Test 1: Basic functionality
$ pytest tests/test_api_performance.py -v

# Test 2: Filter slow tests
$ pytest tests/test_api_performance.py -v -m slow

# Test 3: Exclude slow tests
$ pytest tests/test_api_performance.py -v -m "not slow"

# Test 4: Custom threshold
$ pytest tests/test_api_performance.py -v --slow-threshold=1.5

# Test 5: Show summary
$ pytest tests/test_api_performance.py -v --show-slow-summary
```

**Output for Test 5**:
```
tests/test_api_performance.py::test_fast_operation PASSED               [ 25%]
tests/test_api_performance.py::test_medium_operation PASSED             [ 50%]
tests/test_api_performance.py::test_slow_operation PASSED               [ 75%]
tests/test_api_performance.py::test_very_slow_operation PASSED          [100%]

============================== Slow Tests Summary ==============================
Found 2 test(s) exceeding 0.5s threshold:

  ✓ 2.00s - tests/test_api_performance.py::test_very_slow_operation
  ✓ 1.00s - tests/test_api_performance.py::test_slow_operation

Tip: Run 'pytest -m slow' to run only slow tests

============================== 4 passed in 3.32s ===============================
```

### Common Plugin Patterns

Our slow test plugin demonstrates several common patterns:

#### Pattern 1: Configuration Management

```python
# Support both ini-file and command-line configuration
def pytest_addoption(parser):
    parser.addini("config_name", ...)  # Config file
    parser.addoption("--flag", ...)    # Command-line

def pytest_configure(config):
    # Command-line overrides config file
    value = config.getoption("--flag") or config.getini("config_name")
```

#### Pattern 2: State Management

```python
# Store plugin state in config object
def pytest_configure(config):
    config.plugin_state = []  # Initialize state

def pytest_runtest_makereport(item, call):
    # Access state via item.config
    item.config.plugin_state.append(...)
```

#### Pattern 3: Hook Wrapping

```python
# Wrap around existing hooks to add functionality
@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    outcome = yield  # Let other hooks run
    report = outcome.get_result()  # Get the result
    # Add custom logic here
```

#### Pattern 4: Terminal Output

```python
# Add custom sections to terminal output
def pytest_terminal_summary(terminalreporter, exitstatus, config):
    terminalreporter.section("Custom Section")
    terminalreporter.write_line("Custom output")
```

## Decision Framework: Local vs. Distributed Plugin

| Aspect | Local Plugin (conftest.py) | Distributed Plugin (Package) |
|--------|---------------------------|------------------------------|
| **Scope** | Single project | Multiple projects |
| **Installation** | Automatic (in project) | Manual (pip install) |
| **Maintenance** | Project-specific | Requires versioning |
| **Sharing** | Copy conftest.py | Publish to PyPI |
| **Best for** | Project-specific needs | Reusable functionality |

### When to Keep It Local

Keep your plugin in `conftest.py` when:

1. **Project-specific**: Functionality is unique to your project
2. **Rapid iteration**: You're still experimenting with the design
3. **Simple scope**: Plugin is under 200 lines
4. **Team size**: Small team that doesn't need formal distribution

### When to Distribute

Create a separate package when:

1. **Reusable**: Multiple projects could benefit
2. **Stable**: API is well-defined and unlikely to change frequently
3. **Complex**: Plugin exceeds 200 lines or has multiple modules
4. **Community value**: Others might find it useful

## The Journey: From Problem to Plugin

| Stage | Problem | Solution | Result |
|-------|---------|----------|--------|
| 0 | Slow tests hard to identify | Manual inspection | Time-consuming |
| 1 | Need automatic detection | Hook into test execution | Tests marked automatically |
| 2 | Hardcoded threshold | Add configuration | Flexible threshold |
| 3 | No visibility | Add summary report | Clear actionable output |
| 4 | Configuration scattered | Support pytest.ini | Centralized config |
| 5 | Production use | Add error handling | Robust plugin |

### Lessons Learned

1. **Start simple**: Begin with minimal functionality and iterate
2. **Use hooks wisely**: Understand the hook lifecycle before implementing
3. **Configuration matters**: Support both ini-file and command-line options
4. **State management**: Store plugin state in the config object
5. **User experience**: Provide clear output and helpful error messages
6. **Test your plugin**: Verify it works in various scenarios
7. **Document thoroughly**: Include docstrings and usage examples

## Hooks: How Pytest Plugins Work Under the Hood

## Hooks: How Pytest Plugins Work Under the Hood

We've created a custom plugin, but to truly master plugin development, we need to understand pytest's hook system. Hooks are the foundation of pytest's extensibility—they're the mechanism that allows plugins to modify pytest's behavior at specific points in the test lifecycle.

### The Hook System: A Mental Model

Think of pytest's execution as a journey through a series of checkpoints. At each checkpoint, pytest calls specific hook functions, allowing plugins to:

1. **Observe**: See what's happening (read-only)
2. **Modify**: Change behavior or data
3. **Extend**: Add new functionality

Let's visualize the test execution lifecycle and where hooks are called:

```
Test Session Start
    ↓
pytest_configure()          ← Configure plugins, register markers
    ↓
pytest_collection()         ← Discover test files
    ↓
pytest_collect_file()       ← Process each test file
    ↓
pytest_generate_tests()     ← Generate parametrized tests
    ↓
For each test:
    pytest_runtest_setup()      ← Before test setup
        ↓
    pytest_runtest_call()       ← Run the actual test
        ↓
    pytest_runtest_teardown()   ← After test teardown
        ↓
    pytest_runtest_makereport() ← Create test report
    ↓
pytest_terminal_summary()   ← Display final summary
    ↓
pytest_unconfigure()        ← Cleanup
    ↓
Test Session End
```

### Phase 1: Exploring Hook Execution Order

Let's create a diagnostic plugin that logs every hook call to understand the execution flow:

```python
# conftest.py
import pytest

# Track hook calls
hook_calls = []

def log_hook(hook_name, **kwargs):
    """Log a hook call with its arguments."""
    hook_calls.append({
        "hook": hook_name,
        "args": {k: str(v)[:50] for k, v in kwargs.items()}
    })
    print(f"\n[HOOK] {hook_name}")
    for key, value in kwargs.items():
        print(f"  {key}: {str(value)[:50]}")

# Configuration hooks
def pytest_configure(config):
    """Called after command-line options have been parsed."""
    log_hook("pytest_configure", config=config)

def pytest_sessionstart(session):
    """Called after Session object has been created."""
    log_hook("pytest_sessionstart", session=session)

# Collection hooks
def pytest_collection(session):
    """Called to perform collection."""
    log_hook("pytest_collection", session=session)

def pytest_collectstart(collector):
    """Called before collecting from a collector."""
    log_hook("pytest_collectstart", collector=collector)

def pytest_collect_file(file_path, parent):
    """Called for each file in the test directory."""
    log_hook("pytest_collect_file", file_path=file_path, parent=parent)

# Test execution hooks
def pytest_runtest_setup(item):
    """Called before test setup."""
    log_hook("pytest_runtest_setup", item=item)

def pytest_runtest_call(item):
    """Called to run the test."""
    log_hook("pytest_runtest_call", item=item)

def pytest_runtest_teardown(item):
    """Called after test teardown."""
    log_hook("pytest_runtest_teardown", item=item)

# Reporting hooks
@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Called to create test report."""
    log_hook("pytest_runtest_makereport", item=item, call=call)
    outcome = yield
    report = outcome.get_result()
    print(f"  report.when: {report.when}")
    print(f"  report.outcome: {report.outcome}")

def pytest_terminal_summary(terminalreporter, exitstatus, config):
    """Called to add information to terminal summary."""
    log_hook("pytest_terminal_summary", 
             terminalreporter=terminalreporter,
             exitstatus=exitstatus)

# Cleanup hooks
def pytest_sessionfinish(session, exitstatus):
    """Called after whole test run finished."""
    log_hook("pytest_sessionfinish", session=session, exitstatus=exitstatus)

def pytest_unconfigure(config):
    """Called before test process is exited."""
    log_hook("pytest_unconfigure", config=config)
    
    # Print summary of all hook calls
    print("\n" + "="*70)
    print("HOOK EXECUTION SUMMARY")
    print("="*70)
    for i, call in enumerate(hook_calls, 1):
        print(f"{i}. {call['hook']}")
```

Let's create a simple test to observe the hook execution:

```python
# tests/test_hooks_demo.py
def test_simple():
    """A simple test to observe hook execution."""
    assert 1 + 1 == 2
```

```bash
$ pytest tests/test_hooks_demo.py -v -s
```

**Output** (abbreviated for clarity):
```
[HOOK] pytest_configure
  config: <_pytest.config.Config object at 0x...>

[HOOK] pytest_sessionstart
  session: <Session pytest exitstatus=<UNSET> testsfailed=0>

[HOOK] pytest_collection
  session: <Session pytest exitstatus=<UNSET> testsfailed=0>

[HOOK] pytest_collectstart
  collector: <Dir pytest>

[HOOK] pytest_collect_file
  file_path: tests/test_hooks_demo.py
  parent: <Dir pytest>

[HOOK] pytest_collectstart
  collector: <Module test_hooks_demo.py>

[HOOK] pytest_runtest_setup
  item: <Function test_simple>

[HOOK] pytest_runtest_makereport
  item: <Function test_simple>
  call: <CallInfo when='setup' outcome='passed'>
  report.when: setup
  report.outcome: passed

[HOOK] pytest_runtest_call
  item: <Function test_simple>

[HOOK] pytest_runtest_makereport
  item: <Function test_simple>
  call: <CallInfo when='call' outcome='passed'>
  report.when: call
  report.outcome: passed

[HOOK] pytest_runtest_teardown
  item: <Function test_simple>

[HOOK] pytest_runtest_makereport
  item: <Function test_simple>
  call: <CallInfo when='teardown' outcome='passed'>
  report.when: teardown
  report.outcome: passed

[HOOK] pytest_terminal_summary
  terminalreporter: <_pytest.terminal.TerminalReporter object>
  exitstatus: 0

[HOOK] pytest_sessionfinish
  session: <Session pytest exitstatus=0 testsfailed=0>
  exitstatus: 0

[HOOK] pytest_unconfigure
  config: <_pytest.config.Config object at 0x...>

======================================================================
HOOK EXECUTION SUMMARY
======================================================================
1. pytest_configure
2. pytest_sessionstart
3. pytest_collection
4. pytest_collectstart
5. pytest_collect_file
6. pytest_collectstart
7. pytest_runtest_setup
8. pytest_runtest_makereport
9. pytest_runtest_call
10. pytest_runtest_makereport
11. pytest_runtest_teardown
12. pytest_runtest_makereport
13. pytest_terminal_summary
14. pytest_sessionfinish
15. pytest_unconfigure
```

### Diagnostic Analysis: Understanding Hook Execution

**Key observations**:

1. **Configuration first**: `pytest_configure` runs before anything else
2. **Collection phase**: Multiple hooks for discovering and collecting tests
3. **Three-phase execution**: Each test goes through setup → call → teardown
4. **Three reports**: `pytest_runtest_makereport` is called three times per test
5. **Summary last**: `pytest_terminal_summary` runs after all tests complete

**What this tells us**: Hooks provide fine-grained control over every stage of test execution.

### Phase 2: Hook Implementation Patterns

Now let's explore the three main patterns for implementing hooks:

#### Pattern 1: Simple Hook (Direct Implementation)

The simplest pattern—just implement the hook function:

```python
# conftest.py
def pytest_configure(config):
    """Simple hook - just implement the function."""
    print("Configuring pytest...")
    config.custom_data = {"initialized": True}
```

**When to use**: When you need to execute code at a specific point without modifying pytest's behavior.

#### Pattern 2: Hook Wrapper (Observe and Modify)

Use `@pytest.hookimpl(hookwrapper=True)` to wrap around existing hooks:

```python
# conftest.py
import pytest
import time

@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_call(item):
    """Wrapper hook - measure test execution time."""
    start_time = time.time()
    
    # Let the test run
    outcome = yield
    
    # Code here runs after the test
    duration = time.time() - start_time
    print(f"\n[TIMING] {item.name} took {duration:.4f}s")
    
    # You can access the result
    # outcome.get_result() would return the test result
```

**When to use**: When you need to execute code before AND after an existing hook, or when you need to modify the result.

#### Pattern 3: Hook Specification (First/Last Execution)

Control execution order with `tryfirst` and `trylast`:

```python
# conftest.py
import pytest

@pytest.hookimpl(tryfirst=True)
def pytest_runtest_setup(item):
    """Runs before other setup hooks."""
    print(f"\n[FIRST] Setting up {item.name}")

@pytest.hookimpl(trylast=True)
def pytest_runtest_teardown(item):
    """Runs after other teardown hooks."""
    print(f"\n[LAST] Tearing down {item.name}")
```

**When to use**: When hook execution order matters (e.g., you need to run before/after other plugins).

### Phase 3: Practical Hook Examples

Let's implement several practical plugins using different hooks:

#### Example 1: Test Retry Plugin

Automatically retry flaky tests:

```python
# conftest.py
import pytest

def pytest_addoption(parser):
    """Add retry configuration."""
    parser.addini(
        "retries",
        type="string",
        default="0",
        help="Number of times to retry failed tests"
    )

def pytest_configure(config):
    """Register retry marker."""
    config.addinivalue_line(
        "markers",
        "flaky: mark test as flaky (will be retried on failure)"
    )

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Implement retry logic."""
    outcome = yield
    report = outcome.get_result()
    
    # Only retry on test call failures (not setup/teardown)
    if report.when != "call" or report.outcome != "failed":
        return
    
    # Check if test is marked as flaky
    flaky_marker = item.get_closest_marker("flaky")
    if not flaky_marker:
        return
    
    # Get retry count
    max_retries = int(item.config.getini("retries"))
    if max_retries == 0:
        return
    
    # Track retry attempts
    if not hasattr(item, "retry_count"):
        item.retry_count = 0
    
    if item.retry_count < max_retries:
        item.retry_count += 1
        report.outcome = "rerun"
        report.wasxfail = f"Retry {item.retry_count}/{max_retries}"
```

Test the retry plugin:

```python
# tests/test_retry.py
import pytest
import random

@pytest.mark.flaky
def test_flaky_operation():
    """Test that fails randomly."""
    if random.random() < 0.7:  # 70% chance of failure
        raise AssertionError("Random failure")
    assert True
```

```ini
# pytest.ini
[pytest]
retries = 3
```

```bash
$ pytest tests/test_retry.py -v
```

**Output** (example run):
```
tests/test_retry.py::test_flaky_operation RERUN (Retry 1/3)           [ 50%]
tests/test_retry.py::test_flaky_operation RERUN (Retry 2/3)           [ 50%]
tests/test_retry.py::test_flaky_operation PASSED                      [100%]

============================== 1 passed in 0.03s ===============================
```

#### Example 2: Test Dependency Plugin

Ensure tests run in a specific order:

```python
# conftest.py
import pytest

def pytest_configure(config):
    """Register dependency marker."""
    config.addinivalue_line(
        "markers",
        "depends(name): mark test as dependent on another test"
    )
    config.test_results = {}

@pytest.hookimpl(tryfirst=True)
def pytest_runtest_setup(item):
    """Check dependencies before running test."""
    depends_marker = item.get_closest_marker("depends")
    if not depends_marker:
        return
    
    # Get dependency name
    dependency = depends_marker.args[0]
    
    # Check if dependency passed
    if dependency not in item.config.test_results:
        pytest.skip(f"Dependency '{dependency}' has not run yet")
    
    if item.config.test_results[dependency] != "passed":
        pytest.skip(f"Dependency '{dependency}' did not pass")

@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Store test results for dependency checking."""
    outcome = yield
    report = outcome.get_result()
    
    if report.when == "call":
        item.config.test_results[item.name] = report.outcome
```

Test the dependency plugin:

```python
# tests/test_dependencies.py
import pytest

def test_database_connection():
    """Test that must pass first."""
    assert True  # Simulates successful DB connection

@pytest.mark.depends("test_database_connection")
def test_database_query():
    """Test that depends on database connection."""
    assert True  # Simulates successful query

@pytest.mark.depends("test_database_query")
def test_database_transaction():
    """Test that depends on query working."""
    assert True  # Simulates successful transaction
```

```bash
$ pytest tests/test_dependencies.py -v
```

**Output**:
```
tests/test_dependencies.py::test_database_connection PASSED           [ 33%]
tests/test_dependencies.py::test_database_query PASSED                [ 66%]
tests/test_dependencies.py::test_database_transaction PASSED          [100%]

============================== 3 passed in 0.02s ===============================
```

If the first test fails:

```python
# tests/test_dependencies.py
def test_database_connection():
    """Test that must pass first."""
    assert False  # Simulates failed DB connection
```

```bash
$ pytest tests/test_dependencies.py -v
```

**Output**:
```
tests/test_dependencies.py::test_database_connection FAILED           [ 33%]
tests/test_dependencies.py::test_database_query SKIPPED               [ 66%]
tests/test_dependencies.py::test_database_transaction SKIPPED         [100%]

=================================== FAILURES ===================================
________________________ test_database_connection _____________________________
    def test_database_connection():
        """Test that must pass first."""
>       assert False
E       assert False

tests/test_dependencies.py:5: AssertionError

========================= 1 failed, 2 skipped in 0.03s =========================
```

**What happened**: Dependent tests were automatically skipped because the dependency failed.

### Phase 4: Advanced Hook Techniques

#### Technique 1: Modifying Test Collection

Dynamically modify which tests are collected:

```python
# conftest.py
import pytest

def pytest_collection_modifyitems(config, items):
    """Modify collected test items."""
    # Reorder tests: run fast tests first
    items.sort(key=lambda item: item.get_closest_marker("slow") is not None)
    
    # Add markers based on test name patterns
    for item in items:
        if "integration" in item.nodeid:
            item.add_marker(pytest.mark.integration)
        if "unit" in item.nodeid:
            item.add_marker(pytest.mark.unit)
```

#### Technique 2: Custom Test Outcomes

Create custom test outcomes beyond pass/fail:

```python
# conftest.py
import pytest

def pytest_configure(config):
    """Register custom outcome."""
    config.addinivalue_line(
        "markers",
        "expected_failure: mark test as expected to fail"
    )

@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Handle expected failures."""
    outcome = yield
    report = outcome.get_result()
    
    expected_failure = item.get_closest_marker("expected_failure")
    
    if expected_failure and report.when == "call":
        if report.outcome == "failed":
            report.outcome = "passed"
            report.wasxfail = "Expected failure occurred"
        elif report.outcome == "passed":
            report.outcome = "failed"
            report.longrepr = "Test passed but was expected to fail"
```

#### Technique 3: Fixture Injection via Hooks

Dynamically inject fixtures into tests:

```python
# conftest.py
import pytest

@pytest.fixture
def auto_injected_fixture():
    """Fixture that will be auto-injected."""
    return {"auto": "injected"}

def pytest_generate_tests(metafunc):
    """Automatically inject fixtures into tests."""
    # Inject fixture into all tests in specific modules
    if "auto_test" in metafunc.module.__name__:
        if "auto_injected_fixture" not in metafunc.fixturenames:
            metafunc.fixturenames.append("auto_injected_fixture")
```

### Hook Reference: Most Commonly Used Hooks

| Hook | Purpose | When to Use |
|------|---------|-------------|
| **pytest_configure** | Configure pytest | Register markers, initialize plugin state |
| **pytest_addoption** | Add CLI options | Add custom command-line flags |
| **pytest_collection_modifyitems** | Modify test collection | Reorder tests, add markers dynamically |
| **pytest_generate_tests** | Parametrize tests | Dynamic parametrization, fixture injection |
| **pytest_runtest_setup** | Before test setup | Dependency checking, pre-test validation |
| **pytest_runtest_call** | During test execution | Timing, monitoring |
| **pytest_runtest_teardown** | After test teardown | Cleanup, resource release |
| **pytest_runtest_makereport** | Create test report | Custom outcomes, result tracking |
| **pytest_terminal_summary** | Terminal output | Custom summary sections |

### Hook Execution Order: Complete Reference

For a single test, hooks execute in this order:

1. **Session Start**
   - `pytest_configure`
   - `pytest_sessionstart`

2. **Collection**
   - `pytest_collection`
   - `pytest_collect_file`
   - `pytest_collection_modifyitems`

3. **Test Execution** (per test)
   - `pytest_runtest_protocol` (wrapper)
   - `pytest_runtest_logstart`
   - `pytest_runtest_setup`
   - `pytest_runtest_makereport` (setup phase)
   - `pytest_runtest_call`
   - `pytest_runtest_makereport` (call phase)
   - `pytest_runtest_teardown`
   - `pytest_runtest_makereport` (teardown phase)
   - `pytest_runtest_logfinish`

4. **Session End**
   - `pytest_terminal_summary`
   - `pytest_sessionfinish`
   - `pytest_unconfigure`

## Decision Framework: Which Hook to Use?

| Goal | Hook to Use | Pattern |
|------|-------------|---------|
| **Add configuration** | `pytest_configure` | Simple |
| **Add CLI options** | `pytest_addoption` | Simple |
| **Modify test collection** | `pytest_collection_modifyitems` | Simple |
| **Measure test timing** | `pytest_runtest_call` | Wrapper |
| **Track test results** | `pytest_runtest_makereport` | Wrapper |
| **Add custom output** | `pytest_terminal_summary` | Simple |
| **Implement retries** | `pytest_runtest_makereport` | Wrapper + tryfirst |
| **Check dependencies** | `pytest_runtest_setup` | Simple + tryfirst |

## The Journey: From Hook Confusion to Hook Mastery

| Stage | Understanding | Capability |
|-------|--------------|------------|
| 0 | Hooks are mysterious | Can't extend pytest |
| 1 | Hooks are checkpoints | Can observe execution |
| 2 | Hooks have patterns | Can implement simple plugins |
| 3 | Hooks have order | Can control execution flow |
| 4 | Hooks compose | Can build complex plugins |
| 5 | Hooks are powerful | Can modify pytest behavior completely |

### Lessons Learned

1. **Hooks are checkpoints**: They're called at specific points in pytest's execution
2. **Three main patterns**: Simple, wrapper, and execution order control
3. **Execution order matters**: Use `tryfirst`/`trylast` when order is critical
4. **Wrappers are powerful**: Use `hookwrapper=True` to execute code before and after
5. **State management**: Store plugin state in the config object
6. **Test your hooks**: Verify they work in various scenarios
7. **Document hook usage**: Explain which hooks your plugin uses and why

## Common Plugin Use Cases

## Common Plugin Use Cases

Now that we understand how hooks work, let's explore common real-world scenarios where custom plugins solve practical problems. Each use case demonstrates a different aspect of plugin development and addresses actual pain points in testing workflows.

### Use Case 1: Test Data Management

**Problem**: Tests need consistent, isolated test data, but setting it up manually in each test is repetitive and error-prone.

**Solution**: A plugin that automatically provides fresh test data for each test.

#### Implementation: Auto-Fixture Plugin

```python
# conftest.py
import pytest
import json
from pathlib import Path
from typing import Dict, Any

def pytest_addoption(parser):
    """Add test data configuration."""
    parser.addini(
        "test_data_dir",
        type="string",
        default="tests/data",
        help="Directory containing test data files"
    )

def pytest_configure(config):
    """Load test data at session start."""
    data_dir = Path(config.getini("test_data_dir"))
    config.test_data = {}
    
    if data_dir.exists():
        for data_file in data_dir.glob("*.json"):
            with open(data_file) as f:
                config.test_data[data_file.stem] = json.load(f)

@pytest.fixture
def test_data(request):
    """Provide test data based on test name or marker."""
    # Check for explicit data marker
    data_marker = request.node.get_closest_marker("data")
    if data_marker:
        data_name = data_marker.args[0]
        return request.config.test_data.get(data_name, {})
    
    # Auto-detect based on test name
    test_name = request.node.name
    for data_name, data in request.config.test_data.items():
        if data_name in test_name:
            return data
    
    return {}

def pytest_generate_tests(metafunc):
    """Auto-inject test_data fixture if test data exists."""
    if "test_data" not in metafunc.fixturenames:
        # Check if test name matches any data file
        test_name = metafunc.function.__name__
        for data_name in metafunc.config.test_data.keys():
            if data_name in test_name:
                metafunc.fixturenames.append("test_data")
                break
```

Create test data files:

```json
// tests/data/users.json
{
  "valid_user": {
    "username": "testuser",
    "email": "test@example.com",
    "age": 25
  },
  "invalid_user": {
    "username": "",
    "email": "invalid",
    "age": -1
  }
}
```

```json
// tests/data/products.json
{
  "laptop": {
    "name": "Test Laptop",
    "price": 999.99,
    "stock": 10
  },
  "phone": {
    "name": "Test Phone",
    "price": 599.99,
    "stock": 5
  }
}
```

Use the plugin in tests:

```python
# tests/test_users.py
import pytest

def test_users_valid_user(test_data):
    """Test automatically gets users.json data."""
    user = test_data["valid_user"]
    assert user["username"] == "testuser"
    assert user["age"] == 25

def test_users_invalid_user(test_data):
    """Test automatically gets users.json data."""
    user = test_data["invalid_user"]
    assert user["username"] == ""
    assert user["age"] == -1

@pytest.mark.data("products")
def test_explicit_data_loading(test_data):
    """Test explicitly requests products.json data."""
    laptop = test_data["laptop"]
    assert laptop["price"] == 999.99
```

```bash
$ pytest tests/test_users.py -v
```

**Output**:
```
tests/test_users.py::test_users_valid_user PASSED                     [ 33%]
tests/test_users.py::test_users_invalid_user PASSED                   [ 66%]
tests/test_users.py::test_explicit_data_loading PASSED                [100%]

============================== 3 passed in 0.02s ===============================
```

**What we achieved**:

1. **Automatic data loading**: Test data loads once at session start
2. **Auto-injection**: Tests automatically get relevant data based on naming
3. **Explicit control**: Tests can explicitly request specific data with markers
4. **Centralized management**: All test data in one location

### Use Case 2: Test Environment Validation

**Problem**: Tests fail in CI/CD because required environment variables or dependencies are missing, but the failures are cryptic.

**Solution**: A plugin that validates the test environment before running tests.

#### Implementation: Environment Validator Plugin

```python
# conftest.py
import pytest
import os
import sys
from typing import List, Dict, Any

def pytest_addoption(parser):
    """Add environment validation options."""
    parser.addini(
        "required_env_vars",
        type="linelist",
        help="List of required environment variables"
    )
    parser.addini(
        "required_packages",
        type="linelist",
        help="List of required Python packages"
    )

def pytest_configure(config):
    """Validate environment before running tests."""
    errors = []
    
    # Check required environment variables
    required_vars = config.getini("required_env_vars")
    for var in required_vars:
        if not os.environ.get(var):
            errors.append(f"Missing required environment variable: {var}")
    
    # Check required packages
    required_packages = config.getini("required_packages")
    for package in required_packages:
        try:
            __import__(package)
        except ImportError:
            errors.append(f"Missing required package: {package}")
    
    # Check Python version if specified
    min_python = config.getini("min_python_version")
    if min_python:
        current = sys.version_info[:2]
        required = tuple(map(int, min_python.split(".")))
        if current < required:
            errors.append(
                f"Python {required[0]}.{required[1]}+ required, "
                f"but running {current[0]}.{current[1]}"
            )
    
    if errors:
        error_msg = "\n".join([
            "Environment validation failed:",
            *[f"  - {error}" for error in errors],
            "\nPlease fix these issues before running tests."
        ])
        pytest.exit(error_msg, returncode=1)

def pytest_report_header(config):
    """Add environment information to test report header."""
    return [
        f"Python: {sys.version.split()[0]}",
        f"Platform: {sys.platform}",
        f"Environment: {os.environ.get('TEST_ENV', 'development')}"
    ]
```

Configure environment requirements:

```ini
# pytest.ini
[pytest]
required_env_vars =
    DATABASE_URL
    API_KEY
    TEST_ENV

required_packages =
    requests
    pytest-xdist

min_python_version = 3.8
```

Run tests without required environment:

```bash
$ pytest tests/test_users.py -v
```

**Output**:
```
Environment validation failed:
  - Missing required environment variable: DATABASE_URL
  - Missing required environment variable: API_KEY
  - Missing required environment variable: TEST_ENV

Please fix these issues before running tests.
```

Run tests with proper environment:

```bash
$ DATABASE_URL=postgres://localhost API_KEY=test-key TEST_ENV=ci pytest tests/test_users.py -v
```

**Output**:
```
============================= test session starts ==============================
platform linux -- Python 3.11.0, pytest-7.4.3, pluggy-1.3.0
Python: 3.11.0
Platform: linux
Environment: ci
cachedir: .pytest_cache
rootdir: /home/user/project

tests/test_users.py::test_users_valid_user PASSED                     [ 33%]
tests/test_users.py::test_users_invalid_user PASSED                   [ 66%]
tests/test_users.py::test_explicit_data_loading PASSED                [100%]

============================== 3 passed in 0.02s ===============================
```

### Use Case 3: Test Execution Monitoring

**Problem**: Need to track test execution metrics (timing, memory usage, API calls) for performance analysis.

**Solution**: A plugin that monitors test execution and generates detailed metrics.

#### Implementation: Test Monitor Plugin

```python
# conftest.py
import pytest
import time
import psutil
import os
from typing import Dict, List, Any

class TestMetrics:
    """Store test execution metrics."""
    def __init__(self):
        self.tests: List[Dict[str, Any]] = []
        self.process = psutil.Process(os.getpid())
    
    def start_test(self, item):
        """Record test start metrics."""
        return {
            "name": item.nodeid,
            "start_time": time.time(),
            "start_memory": self.process.memory_info().rss / 1024 / 1024,  # MB
        }
    
    def end_test(self, metrics, outcome):
        """Record test end metrics."""
        metrics["end_time"] = time.time()
        metrics["end_memory"] = self.process.memory_info().rss / 1024 / 1024
        metrics["duration"] = metrics["end_time"] - metrics["start_time"]
        metrics["memory_delta"] = metrics["end_memory"] - metrics["start_memory"]
        metrics["outcome"] = outcome
        self.tests.append(metrics)

def pytest_configure(config):
    """Initialize metrics tracking."""
    config.test_metrics = TestMetrics()

@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_protocol(item, nextitem):
    """Monitor test execution."""
    metrics = item.config.test_metrics.start_test(item)
    
    outcome = yield
    
    # Get test outcome
    reports = [rep for rep in item.stash.values() if hasattr(rep, 'outcome')]
    test_outcome = reports[-1].outcome if reports else "unknown"
    
    item.config.test_metrics.end_test(metrics, test_outcome)

def pytest_terminal_summary(terminalreporter, exitstatus, config):
    """Display metrics summary."""
    metrics = config.test_metrics
    
    if not metrics.tests:
        return
    
    terminalreporter.section("Test Execution Metrics")
    
    # Calculate statistics
    total_duration = sum(t["duration"] for t in metrics.tests)
    total_memory = sum(t["memory_delta"] for t in metrics.tests)
    avg_duration = total_duration / len(metrics.tests)
    
    terminalreporter.write_line(
        f"\nTotal tests: {len(metrics.tests)}"
    )
    terminalreporter.write_line(
        f"Total duration: {total_duration:.2f}s"
    )
    terminalreporter.write_line(
        f"Average duration: {avg_duration:.2f}s"
    )
    terminalreporter.write_line(
        f"Total memory delta: {total_memory:.2f} MB\n"
    )
    
    # Show slowest tests
    slowest = sorted(metrics.tests, key=lambda x: x["duration"], reverse=True)[:5]
    terminalreporter.write_line("Slowest tests:")
    for test in slowest:
        terminalreporter.write_line(
            f"  {test['duration']:.2f}s - {test['name']}"
        )
    
    # Show memory-intensive tests
    memory_intensive = sorted(
        metrics.tests, 
        key=lambda x: abs(x["memory_delta"]), 
        reverse=True
    )[:5]
    terminalreporter.write_line("\nMemory-intensive tests:")
    for test in memory_intensive:
        terminalreporter.write_line(
            f"  {test['memory_delta']:+.2f} MB - {test['name']}"
        )
```

Run tests with monitoring:

```bash
$ pytest tests/test_api_performance.py -v
```

**Output**:
```
tests/test_api_performance.py::test_fast_operation PASSED             [ 25%]
tests/test_api_performance.py::test_medium_operation PASSED           [ 50%]
tests/test_api_performance.py::test_slow_operation PASSED             [ 75%]
tests/test_api_performance.py::test_very_slow_operation PASSED        [100%]

============================== Test Execution Metrics ==========================

Total tests: 4
Total duration: 3.32s
Average duration: 0.83s
Total memory delta: 0.15 MB

Slowest tests:
  2.00s - tests/test_api_performance.py::test_very_slow_operation
  1.00s - tests/test_api_performance.py::test_slow_operation
  0.30s - tests/test_api_performance.py::test_medium_operation
  0.01s - tests/test_api_performance.py::test_fast_operation

Memory-intensive tests:
  +0.05 MB - tests/test_api_performance.py::test_very_slow_operation
  +0.04 MB - tests/test_api_performance.py::test_slow_operation
  +0.03 MB - tests/test_api_performance.py::test_medium_operation
  +0.03 MB - tests/test_api_performance.py::test_fast_operation

============================== 4 passed in 3.32s ===============================
```

### Use Case 4: Test Result Notification

**Problem**: Need to notify team members when tests fail in CI/CD, but don't want to parse pytest output manually.

**Solution**: A plugin that sends notifications to Slack, email, or other services when tests fail.

#### Implementation: Notification Plugin

```python
# conftest.py
import pytest
import json
import os
from typing import List, Dict, Any
from urllib.request import Request, urlopen
from urllib.error import URLError

def pytest_addoption(parser):
    """Add notification options."""
    parser.addini(
        "slack_webhook_url",
        type="string",
        help="Slack webhook URL for notifications"
    )
    parser.addini(
        "notify_on_failure",
        type="bool",
        default=True,
        help="Send notification when tests fail"
    )

def pytest_configure(config):
    """Initialize notification state."""
    config.failed_tests = []
    config.passed_tests = []

@pytest.hookimpl(hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Track test results."""
    outcome = yield
    report = outcome.get_result()
    
    if report.when == "call":
        if report.outcome == "passed":
            item.config.passed_tests.append(item.nodeid)
        elif report.outcome == "failed":
            item.config.failed_tests.append({
                "name": item.nodeid,
                "error": str(report.longrepr)[:200]  # Truncate long errors
            })

def pytest_sessionfinish(session, exitstatus):
    """Send notification at end of test session."""
    config = session.config
    
    # Only notify on failure if configured
    if not config.getini("notify_on_failure"):
        return
    
    if not config.failed_tests:
        return
    
    # Get webhook URL
    webhook_url = config.getini("slack_webhook_url")
    if not webhook_url:
        webhook_url = os.environ.get("SLACK_WEBHOOK_URL")
    
    if not webhook_url:
        return
    
    # Prepare notification message
    total_tests = len(config.passed_tests) + len(config.failed_tests)
    message = {
        "text": f"❌ Test Suite Failed",
        "blocks": [
            {
                "type": "header",
                "text": {
                    "type": "plain_text",
                    "text": "❌ Test Suite Failed"
                }
            },
            {
                "type": "section",
                "fields": [
                    {
                        "type": "mrkdwn",
                        "text": f"*Total Tests:*\n{total_tests}"
                    },
                    {
                        "type": "mrkdwn",
                        "text": f"*Failed:*\n{len(config.failed_tests)}"
                    },
                    {
                        "type": "mrkdwn",
                        "text": f"*Passed:*\n{len(config.passed_tests)}"
                    },
                    {
                        "type": "mrkdwn",
                        "text": f"*Environment:*\n{os.environ.get('TEST_ENV', 'unknown')}"
                    }
                ]
            }
        ]
    }
    
    # Add failed test details
    if config.failed_tests:
        failed_text = "\n".join([
            f"• {test['name']}"
            for test in config.failed_tests[:5]  # Limit to 5
        ])
        if len(config.failed_tests) > 5:
            failed_text += f"\n... and {len(config.failed_tests) - 5} more"
        
        message["blocks"].append({
            "type": "section",
            "text": {
                "type": "mrkdwn",
                "text": f"*Failed Tests:*\n{failed_text}"
            }
        })
    
    # Send notification
    try:
        req = Request(
            webhook_url,
            data=json.dumps(message).encode('utf-8'),
            headers={'Content-Type': 'application/json'}
        )
        with urlopen(req, timeout=10) as response:
            if response.status != 200:
                print(f"Failed to send notification: {response.status}")
    except URLError as e:
        print(f"Failed to send notification: {e}")
```

Configure notification:

```ini
# pytest.ini
[pytest]
notify_on_failure = true
slack_webhook_url = https://hooks.slack.com/services/YOUR/WEBHOOK/URL
```

Or use environment variable:

```bash
$ export SLACK_WEBHOOK_URL=https://hooks.slack.com/services/YOUR/WEBHOOK/URL
$ pytest tests/
```

When tests fail, the team receives a Slack notification with:
- Total test count
- Number of failures
- List of failed tests
- Environment information

### Use Case 5: Test Coverage Enforcement

**Problem**: Want to enforce minimum coverage thresholds and fail the build if coverage drops below acceptable levels.

**Solution**: A plugin that integrates with pytest-cov and enforces coverage rules.

#### Implementation: Coverage Enforcer Plugin

```python
# conftest.py
import pytest
from typing import Dict

def pytest_addoption(parser):
    """Add coverage enforcement options."""
    parser.addini(
        "min_coverage",
        type="string",
        default="80",
        help="Minimum coverage percentage required"
    )
    parser.addini(
        "coverage_per_file",
        type="bool",
        default=False,
        help="Enforce coverage per file"
    )

def pytest_terminal_summary(terminalreporter, exitstatus, config):
    """Enforce coverage requirements."""
    # Check if pytest-cov is active
    if not hasattr(config, '_cov'):
        return
    
    min_coverage = float(config.getini("min_coverage"))
    per_file = config.getini("coverage_per_file")
    
    # Get coverage data
    cov = config._cov.cov
    total = cov.report(file=None)
    
    terminalreporter.section("Coverage Enforcement")
    
    # Check total coverage
    if total < min_coverage:
        terminalreporter.write_line(
            f"❌ Total coverage {total:.1f}% is below minimum {min_coverage}%",
            red=True
        )
        pytest.exit(
            f"Coverage {total:.1f}% is below minimum {min_coverage}%",
            returncode=1
        )
    else:
        terminalreporter.write_line(
            f"✓ Total coverage {total:.1f}% meets minimum {min_coverage}%",
            green=True
        )
    
    # Check per-file coverage if enabled
    if per_file:
        files_below_threshold = []
        for filename in cov.get_data().measured_files():
            analysis = cov.analysis(filename)
            file_coverage = (
                len(analysis.executed) / len(analysis.statements) * 100
                if analysis.statements else 100
            )
            if file_coverage < min_coverage:
                files_below_threshold.append((filename, file_coverage))
        
        if files_below_threshold:
            terminalreporter.write_line(
                f"\n❌ {len(files_below_threshold)} file(s) below {min_coverage}% coverage:",
                red=True
            )
            for filename, coverage in files_below_threshold:
                terminalreporter.write_line(
                    f"  {coverage:.1f}% - {filename}"
                )
            pytest.exit(
                f"{len(files_below_threshold)} files below coverage threshold",
                returncode=1
            )
```

Configure coverage enforcement:

```ini
# pytest.ini
[pytest]
min_coverage = 80
coverage_per_file = true
addopts = --cov=src --cov-report=term-missing
```

```bash
$ pytest tests/ -v
```

**Output** (if coverage is below threshold):
```
============================== Coverage Enforcement ============================
❌ Total coverage 65.3% is below minimum 80.0%

Coverage 65.3% is below minimum 80.0%
```

**Output** (if coverage meets threshold):
```
============================== Coverage Enforcement ============================
✓ Total coverage 85.7% meets minimum 80.0%

============================== 10 passed in 2.15s ===============================
```

## Decision Framework: When to Create a Plugin

| Scenario | Plugin? | Alternative |
|----------|---------|-------------|
| **Reusable across projects** | ✅ Yes | - |
| **Project-specific logic** | ✅ Yes (local) | conftest.py |
| **One-time use** | ❌ No | Test helper function |
| **Modifies pytest behavior** | ✅ Yes | - |
| **Simple fixture** | ❌ No | Regular fixture |
| **Complex workflow** | ✅ Yes | - |
| **Team needs it** | ✅ Yes | - |

## Common Plugin Patterns Summary

| Pattern | Use Case | Example |
|---------|----------|---------|
| **Auto-fixture** | Automatic test data injection | Test data management |
| **Validator** | Pre-test environment checks | Environment validation |
| **Monitor** | Track execution metrics | Performance monitoring |
| **Notifier** | External integrations | Slack notifications |
| **Enforcer** | Quality gates | Coverage enforcement |

## The Journey: From Manual to Automated

| Stage | Approach | Efficiency |
|-------|----------|-----------|
| 0 | Manual test data setup | Low |
| 1 | Shared fixtures | Medium |
| 2 | Auto-injection plugin | High |
| 3 | Environment validation | Higher |
| 4 | Monitoring + notifications | Highest |

### Lessons Learned

1. **Plugins solve repetitive problems**: If you're doing something manually in every test, consider a plugin
2. **Start local, distribute later**: Begin with conftest.py, extract to package when needed
3. **Configuration is key**: Make plugins configurable for different environments
4. **Fail fast**: Validate environment before running tests
5. **Provide visibility**: Add custom terminal sections for important information
6. **Integrate with tools**: Plugins can bridge pytest with external services
7. **Think about the team**: Plugins should make everyone's life easier

## Distributing Your Own Plugin

## Distributing Your Own Plugin

You've created a useful plugin that solves a real problem. Now you want to share it with the community or use it across multiple projects. Let's learn how to package and distribute a pytest plugin properly.

### Phase 1: From conftest.py to Package

Our slow test detection plugin has been working well in `conftest.py`. Let's transform it into a distributable package.

#### Step 1: Create Package Structure

First, create a proper package structure:

```bash
$ mkdir pytest-slow-detector
$ cd pytest-slow-detector
$ mkdir pytest_slow_detector
$ touch pytest_slow_detector/__init__.py
$ touch pytest_slow_detector/plugin.py
$ touch setup.py
$ touch README.md
$ touch LICENSE
```

**Directory structure**:
```
pytest-slow-detector/
├── pytest_slow_detector/
│   ├── __init__.py
│   └── plugin.py
├── setup.py
├── README.md
├── LICENSE
└── tests/
    └── test_plugin.py
```

#### Step 2: Move Plugin Code

Move the plugin code from `conftest.py` to `plugin.py`:

```python
# pytest_slow_detector/plugin.py
"""
Pytest plugin for automatic slow test detection and reporting.
"""
import pytest
from typing import Dict, List, Any

__version__ = "0.1.0"

def pytest_addoption(parser):
    """Add custom command-line options and ini-file values."""
    parser.addini(
        "slow_threshold",
        type="string",
        default="0.5",
        help="Threshold in seconds for marking tests as slow (default: 0.5)"
    )
    parser.addini(
        "show_slow_summary",
        type="bool",
        default=False,
        help="Show summary of slow tests at the end (default: False)"
    )
    
    parser.addoption(
        "--slow-threshold",
        action="store",
        type=float,
        help="Override slow_threshold from config file"
    )

def pytest_configure(config):
    """Register custom markers and initialize plugin state."""
    config.addinivalue_line(
        "markers",
        "slow: marks tests as slow (automatically applied based on duration)"
    )
    
    # Get threshold from command-line or config file
    threshold = config.getoption("--slow-threshold")
    if threshold is None:
        try:
            threshold = float(config.getini("slow_threshold"))
        except (ValueError, TypeError):
            threshold = 0.5
    
    if threshold < 0:
        raise pytest.UsageError(
            f"slow_threshold must be non-negative, got {threshold}"
        )
    
    config.slow_threshold = threshold
    config.slow_tests: List[Dict[str, Any]] = []

@pytest.hookimpl(tryfirst=True, hookwrapper=True)
def pytest_runtest_makereport(item, call):
    """Automatically mark slow tests and collect information."""
    outcome = yield
    report = outcome.get_result()
    
    if report.when != "call":
        return
    
    threshold = item.config.slow_threshold
    
    if report.duration > threshold:
        item.add_marker(pytest.mark.slow)
        item.config.slow_tests.append({
            "nodeid": item.nodeid,
            "duration": report.duration,
            "threshold": threshold,
            "outcome": report.outcome
        })

def pytest_terminal_summary(terminalreporter, exitstatus, config):
    """Add custom section to terminal summary showing slow tests."""
    show_summary = config.getini("show_slow_summary")
    if not show_summary:
        return
    
    slow_tests = config.slow_tests
    if not slow_tests:
        return
    
    slow_tests.sort(key=lambda x: x["duration"], reverse=True)
    
    terminalreporter.section("Slow Tests Summary")
    terminalreporter.write_line(
        f"Found {len(slow_tests)} test(s) exceeding "
        f"{config.slow_threshold}s threshold:\n"
    )
    
    for test in slow_tests:
        status_symbol = "✓" if test["outcome"] == "passed" else "✗"
        terminalreporter.write_line(
            f"  {status_symbol} {test['duration']:.2f}s - {test['nodeid']}"
        )
    
    terminalreporter.write_line(
        f"\nTip: Run 'pytest -m slow' to run only slow tests"
    )
```

#### Step 3: Create Package Entry Point

Update `__init__.py` to expose the plugin:

```python
# pytest_slow_detector/__init__.py
"""Pytest plugin for automatic slow test detection."""

from .plugin import __version__

__all__ = ["__version__"]
```

#### Step 4: Create setup.py

The `setup.py` file is crucial—it tells pip how to install your plugin and registers it with pytest:

```python
# setup.py
from setuptools import setup, find_packages

with open("README.md", "r", encoding="utf-8") as fh:
    long_description = fh.read()

setup(
    name="pytest-slow-detector",
    version="0.1.0",
    author="Your Name",
    author_email="your.email@example.com",
    description="Pytest plugin for automatic slow test detection",
    long_description=long_description,
    long_description_content_type="text/markdown",
    url="https://github.com/yourusername/pytest-slow-detector",
    packages=find_packages(),
    classifiers=[
        "Development Status :: 3 - Alpha",
        "Framework :: Pytest",
        "Intended Audience :: Developers",
        "License :: OSI Approved :: MIT License",
        "Operating System :: OS Independent",
        "Programming Language :: Python :: 3",
        "Programming Language :: Python :: 3.8",
        "Programming Language :: Python :: 3.9",
        "Programming Language :: Python :: 3.10",
        "Programming Language :: Python :: 3.11",
        "Topic :: Software Development :: Testing",
    ],
    python_requires=">=3.8",
    install_requires=[
        "pytest>=7.0.0",
    ],
    # This is the critical part - registers the plugin with pytest
    entry_points={
        "pytest11": [
            "slow_detector = pytest_slow_detector.plugin",
        ],
    },
)
```

**Key elements**:

1. **entry_points**: Registers the plugin with pytest using the `pytest11` entry point
2. **install_requires**: Lists pytest as a dependency
3. **classifiers**: Helps users find your plugin on PyPI
4. **python_requires**: Specifies minimum Python version

### Phase 2: Testing Your Plugin

Before distributing, thoroughly test your plugin:

```python
# tests/test_plugin.py
"""Tests for pytest-slow-detector plugin."""
import pytest

pytest_plugins = ["pytest_slow_detector.plugin"]

def test_slow_marker_applied(testdir):
    """Test that slow marker is automatically applied."""
    testdir.makepyfile("""
        import time
        
        def test_fast():
            assert True
        
        def test_slow():
            time.sleep(0.6)
            assert True
    """)
    
    testdir.makeini("""
        [pytest]
        slow_threshold = 0.5
    """)
    
    result = testdir.runpytest("-v", "-m", "slow")
    result.stdout.fnmatch_lines([
        "*test_slow PASSED*",
    ])
    assert result.ret == 0

def test_custom_threshold(testdir):
    """Test custom threshold configuration."""
    testdir.makepyfile("""
        import time
        
        def test_medium():
            time.sleep(0.8)
            assert True
    """)
    
    result = testdir.runpytest("-v", "--slow-threshold=1.0", "-m", "slow")
    # Should not match because 0.8s < 1.0s threshold
    assert "test_medium" not in result.stdout.str()

def test_summary_display(testdir):
    """Test that summary is displayed when configured."""
    testdir.makepyfile("""
        import time
        
        def test_slow():
            time.sleep(0.6)
            assert True
    """)
    
    testdir.makeini("""
        [pytest]
        slow_threshold = 0.5
        show_slow_summary = true
    """)
    
    result = testdir.runpytest("-v")
    result.stdout.fnmatch_lines([
        "*Slow Tests Summary*",
        "*Found 1 test(s) exceeding 0.5s threshold*",
    ])

def test_negative_threshold_error(testdir):
    """Test that negative threshold raises error."""
    testdir.makepyfile("""
        def test_anything():
            assert True
    """)
    
    result = testdir.runpytest("--slow-threshold=-1.0")
    assert result.ret != 0
    assert "slow_threshold must be non-negative" in result.stdout.str()
```

Run the plugin tests:

```bash
$ pytest tests/test_plugin.py -v
```

**Output**:
```
tests/test_plugin.py::test_slow_marker_applied PASSED                 [ 25%]
tests/test_plugin.py::test_custom_threshold PASSED                    [ 50%]
tests/test_plugin.py::test_summary_display PASSED                     [ 75%]
tests/test_plugin.py::test_negative_threshold_error PASSED            [100%]

============================== 4 passed in 1.23s ===============================
```

### Phase 3: Documentation

Create comprehensive documentation in `README.md`:

```markdown
# pytest-slow-detector

Pytest plugin for automatic slow test detection and reporting.

## Features

- Automatically marks tests exceeding a configurable duration threshold
- Provides filtering capabilities with `-m slow` and `-m "not slow"`
- Generates summary report of slow tests
- Configurable via pytest.ini or command-line

## Installation

```bash
pip install pytest-slow-detector
```

## Usage

### Basic Usage

```bash
# Run all tests and see slow test summary
pytest --show-slow-summary

# Run only slow tests
pytest -m slow

# Skip slow tests
pytest -m "not slow"
```

### Configuration

Configure in `pytest.ini`:

```ini
[pytest]
slow_threshold = 0.5  # seconds
show_slow_summary = true
```

Or use command-line:

```bash
pytest --slow-threshold=1.0
```

## Examples

### Example 1: Identify Slow Tests

```python
# tests/test_api.py
def test_fast_endpoint():
    response = client.get("/fast")
    assert response.status_code == 200

def test_slow_endpoint():
    response = client.get("/slow")  # Takes 2 seconds
    assert response.status_code == 200
```

Run with summary:

```bash
$ pytest --show-slow-summary

============================== Slow Tests Summary ==============================
Found 1 test(s) exceeding 0.5s threshold:

  ✓ 2.00s - tests/test_api.py::test_slow_endpoint

Tip: Run 'pytest -m slow' to run only slow tests
```

### Example 2: Skip Slow Tests in CI

```bash
# In CI pipeline, skip slow tests for fast feedback
pytest -m "not slow"
```

## Configuration Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `slow_threshold` | float | 0.5 | Threshold in seconds for marking tests as slow |
| `show_slow_summary` | bool | false | Show summary of slow tests at the end |
| `--slow-threshold` | float | - | Override threshold from command-line |

## Requirements

- Python 3.8+
- pytest 7.0.0+

## License

MIT License - see LICENSE file for details.

## Contributing

Contributions welcome! Please open an issue or submit a pull request.

## Changelog

### 0.1.0 (2024-01-15)

- Initial release
- Automatic slow test detection
- Summary reporting
- Configurable threshold
```

### Phase 4: Local Installation and Testing

Before publishing, test the installation locally:

```bash
# Install in development mode
$ pip install -e .

# Verify installation
$ pip list | grep pytest-slow-detector
pytest-slow-detector    0.1.0    /path/to/pytest-slow-detector

# Test in a real project
$ cd /path/to/your/project
$ pytest --help | grep slow
  --slow-threshold=SLOW_THRESHOLD
                        Override slow_threshold from config file
```

### Phase 5: Publishing to PyPI

#### Step 1: Create PyPI Account

1. Go to https://pypi.org and create an account
2. Verify your email
3. Enable two-factor authentication (recommended)

#### Step 2: Install Build Tools

```bash
$ pip install build twine
```

#### Step 3: Build Distribution

```bash
$ python -m build
```

**Output**:
```
* Creating venv isolated environment...
* Installing packages in isolated environment... (setuptools>=40.8.0, wheel)
* Getting build dependencies for sdist...
* Building sdist...
* Building wheel from sdist
* Creating venv isolated environment...
* Installing packages in isolated environment... (setuptools>=40.8.0, wheel)
* Getting build dependencies for wheel...
* Building wheel...
Successfully built pytest-slow-detector-0.1.0.tar.gz and pytest_slow_detector-0.1.0-py3-none-any.whl
```

This creates two files in the `dist/` directory:
- `pytest-slow-detector-0.1.0.tar.gz` (source distribution)
- `pytest_slow_detector-0.1.0-py3-none-any.whl` (wheel distribution)

#### Step 4: Test on TestPyPI First

Before publishing to the real PyPI, test on TestPyPI:

```bash
# Upload to TestPyPI
$ python -m twine upload --repository testpypi dist/*

# Test installation from TestPyPI
$ pip install --index-url https://test.pypi.org/simple/ pytest-slow-detector
```

#### Step 5: Publish to PyPI

```bash
$ python -m twine upload dist/*
```

**Output**:
```
Uploading distributions to https://upload.pypi.org/legacy/
Enter your username: yourusername
Enter your password: 
Uploading pytest_slow_detector-0.1.0-py3-none-any.whl
100% ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 12.3/12.3 kB • 00:00 • ?
Uploading pytest-slow-detector-0.1.0.tar.gz
100% ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━ 8.9/8.9 kB • 00:00 • ?

View at:
https://pypi.org/project/pytest-slow-detector/0.1.0/
```

Your plugin is now publicly available!

### Phase 6: Maintenance and Updates

#### Versioning Strategy

Follow semantic versioning (MAJOR.MINOR.PATCH):

- **MAJOR**: Breaking changes (e.g., 1.0.0 → 2.0.0)
- **MINOR**: New features, backward compatible (e.g., 1.0.0 → 1.1.0)
- **PATCH**: Bug fixes (e.g., 1.0.0 → 1.0.1)

#### Releasing Updates

```bash
# 1. Update version in setup.py and plugin.py
# 2. Update CHANGELOG.md
# 3. Commit changes
$ git add .
$ git commit -m "Release version 0.2.0"
$ git tag v0.2.0
$ git push origin main --tags

# 4. Build and upload
$ python -m build
$ python -m twine upload dist/*
```

### Modern Alternative: pyproject.toml

For modern Python projects, use `pyproject.toml` instead of `setup.py`:

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "pytest-slow-detector"
version = "0.1.0"
description = "Pytest plugin for automatic slow test detection"
readme = "README.md"
authors = [
    {name = "Your Name", email = "your.email@example.com"}
]
license = {text = "MIT"}
classifiers = [
    "Development Status :: 3 - Alpha",
    "Framework :: Pytest",
    "Intended Audience :: Developers",
    "License :: OSI Approved :: MIT License",
    "Operating System :: OS Independent",
    "Programming Language :: Python :: 3",
    "Programming Language :: Python :: 3.8",
    "Programming Language :: Python :: 3.9",
    "Programming Language :: Python :: 3.10",
    "Programming Language :: Python :: 3.11",
    "Topic :: Software Development :: Testing",
]
requires-python = ">=3.8"
dependencies = [
    "pytest>=7.0.0",
]

[project.urls]
Homepage = "https://github.com/yourusername/pytest-slow-detector"
Documentation = "https://github.com/yourusername/pytest-slow-detector#readme"
Repository = "https://github.com/yourusername/pytest-slow-detector"
Issues = "https://github.com/yourusername/pytest-slow-detector/issues"

[project.entry-points.pytest11]
slow_detector = "pytest_slow_detector.plugin"

[tool.setuptools.packages.find]
where = ["."]
include = ["pytest_slow_detector*"]
```

### Plugin Distribution Checklist

Before publishing your plugin, ensure:

- [ ] **Code quality**
  - [ ] All tests pass
  - [ ] Code is well-documented
  - [ ] No hardcoded values
  - [ ] Error handling is robust

- [ ] **Documentation**
  - [ ] README.md with examples
  - [ ] Installation instructions
  - [ ] Configuration options documented
  - [ ] Changelog maintained

- [ ] **Package metadata**
  - [ ] Correct version number
  - [ ] Accurate dependencies
  - [ ] Proper classifiers
  - [ ] License file included

- [ ] **Testing**
  - [ ] Plugin tests pass
  - [ ] Tested in real projects
  - [ ] Tested on TestPyPI
  - [ ] Multiple Python versions tested

- [ ] **Repository**
  - [ ] Code on GitHub/GitLab
  - [ ] Issues enabled
  - [ ] Contributing guidelines
  - [ ] CI/CD configured

## Decision Framework: Distribution Strategy

| Scenario | Strategy | Rationale |
|----------|----------|-----------|
| **Internal company use** | Private PyPI or Git | Security, control |
| **Open source** | Public PyPI | Community access |
| **Experimental** | TestPyPI only | Safe testing |
| **Single project** | Keep in conftest.py | Simplicity |
| **Multiple projects** | Private package | Reusability |
| **Community value** | Public PyPI + GitHub | Maximum impact |

## The Journey: From Local to Public

| Stage | Scope | Distribution |
|-------|-------|--------------|
| 0 | Single test file | Inline code |
| 1 | Single project | conftest.py |
| 2 | Multiple projects (internal) | Private package |
| 3 | Company-wide | Private PyPI |
| 4 | Open source | Public PyPI |
| 5 | Popular plugin | Featured on pytest.org |

### Lessons Learned

1. **Start local**: Develop in conftest.py first, extract when proven useful
2. **Test thoroughly**: Plugin bugs affect all users, not just one project
3. **Document well**: Good documentation is as important as good code
4. **Version carefully**: Follow semantic versioning strictly
5. **Test on TestPyPI**: Always test distribution before publishing to PyPI
6. **Maintain actively**: Respond to issues, update for new pytest versions
7. **Community matters**: Engage with users, accept contributions
8. **Keep it simple**: Plugins should solve one problem well
