# Chapter 16: Continuous Integration and Automation

## Running Tests in CI/CD Pipelines

## The Reality of Manual Testing

You've built a comprehensive test suite. Your tests pass locally. You commit your code, push to the repository, and... three days later, a teammate discovers your changes broke the production deployment. The tests still pass on your machine, but the production environment uses Python 3.11 while you develop on 3.9. A dependency version mismatch causes silent failures.

This is the problem Continuous Integration (CI) solves: **automated, consistent test execution in a controlled environment on every code change**.

### What Is CI/CD?

**Continuous Integration (CI)**: Automatically running tests every time code is pushed to a repository. The goal is to catch integration problems early, before they reach production.

**Continuous Deployment/Delivery (CD)**: Automatically deploying code that passes all tests. We'll focus on the CI aspect—ensuring your tests run reliably in an automated environment.

### The Reference Implementation: A Payment Processing System

Throughout this chapter, we'll work with a realistic payment processing system that needs to run in CI. This system will expose the real challenges of automated testing: environment differences, dependency management, test isolation, and reporting.

```python
# src/payment_processor.py
import os
from decimal import Decimal
from typing import Optional
import requests

class PaymentGateway:
    """Simulates interaction with a payment gateway API."""
    
    def __init__(self, api_key: str, base_url: str):
        self.api_key = api_key
        self.base_url = base_url
    
    def charge(self, amount: Decimal, card_token: str) -> dict:
        """Process a payment charge."""
        response = requests.post(
            f"{self.base_url}/charges",
            json={
                "amount": str(amount),
                "card_token": card_token,
                "api_key": self.api_key
            },
            timeout=5
        )
        response.raise_for_status()
        return response.json()

class PaymentProcessor:
    """Business logic for processing payments."""
    
    def __init__(self, gateway: PaymentGateway, min_amount: Decimal = Decimal("0.50")):
        self.gateway = gateway
        self.min_amount = min_amount
    
    def process_payment(self, amount: Decimal, card_token: str) -> dict:
        """
        Process a payment with validation.
        
        Returns:
            dict with keys: success (bool), transaction_id (str), message (str)
        """
        if amount < self.min_amount:
            return {
                "success": False,
                "transaction_id": None,
                "message": f"Amount must be at least {self.min_amount}"
            }
        
        try:
            result = self.gateway.charge(amount, card_token)
            return {
                "success": True,
                "transaction_id": result["id"],
                "message": "Payment processed successfully"
            }
        except requests.RequestException as e:
            return {
                "success": False,
                "transaction_id": None,
                "message": f"Payment gateway error: {str(e)}"
            }

def create_processor_from_env() -> PaymentProcessor:
    """Factory function that reads configuration from environment."""
    api_key = os.environ.get("PAYMENT_API_KEY")
    if not api_key:
        raise ValueError("PAYMENT_API_KEY environment variable not set")
    
    base_url = os.environ.get("PAYMENT_API_URL", "https://api.payment-gateway.example.com")
    
    gateway = PaymentGateway(api_key, base_url)
    return PaymentProcessor(gateway)
```

### Initial Test Suite

Our test suite uses mocking to avoid real API calls and fixtures for test data:

```python
# tests/test_payment_processor.py
from decimal import Decimal
from unittest.mock import Mock, patch
import pytest
from src.payment_processor import PaymentProcessor, PaymentGateway, create_processor_from_env

@pytest.fixture
def mock_gateway():
    """Create a mock payment gateway."""
    gateway = Mock(spec=PaymentGateway)
    gateway.charge.return_value = {"id": "txn_12345", "status": "succeeded"}
    return gateway

@pytest.fixture
def processor(mock_gateway):
    """Create a payment processor with mocked gateway."""
    return PaymentProcessor(mock_gateway)

def test_successful_payment(processor, mock_gateway):
    """Test processing a valid payment."""
    result = processor.process_payment(Decimal("10.00"), "card_tok_visa")
    
    assert result["success"] is True
    assert result["transaction_id"] == "txn_12345"
    assert "successfully" in result["message"]
    
    mock_gateway.charge.assert_called_once_with(Decimal("10.00"), "card_tok_visa")

def test_payment_below_minimum(processor, mock_gateway):
    """Test payment amount below minimum threshold."""
    result = processor.process_payment(Decimal("0.25"), "card_tok_visa")
    
    assert result["success"] is False
    assert result["transaction_id"] is None
    assert "at least" in result["message"]
    
    mock_gateway.charge.assert_not_called()

def test_gateway_error_handling(processor, mock_gateway):
    """Test handling of gateway API errors."""
    import requests
    mock_gateway.charge.side_effect = requests.RequestException("Network timeout")
    
    result = processor.process_payment(Decimal("10.00"), "card_tok_visa")
    
    assert result["success"] is False
    assert result["transaction_id"] is None
    assert "gateway error" in result["message"]

@patch.dict("os.environ", {"PAYMENT_API_KEY": "test_key_123"})
def test_factory_creates_processor():
    """Test factory function reads from environment."""
    processor = create_processor_from_env()
    assert processor is not None
    assert processor.gateway.api_key == "test_key_123"

def test_factory_requires_api_key():
    """Test factory fails without API key."""
    import os
    # Ensure key is not set
    os.environ.pop("PAYMENT_API_KEY", None)
    
    with pytest.raises(ValueError, match="PAYMENT_API_KEY"):
        create_processor_from_env()
```

### Running Tests Locally

On your development machine, these tests pass:

```bash
$ pytest tests/test_payment_processor.py -v
======================== test session starts =========================
collected 5 items

tests/test_payment_processor.py::test_successful_payment PASSED  [ 20%]
tests/test_payment_processor.py::test_payment_below_minimum PASSED [ 40%]
tests/test_payment_processor.py::test_gateway_error_handling PASSED [ 60%]
tests/test_payment_processor.py::test_factory_creates_processor PASSED [ 80%]
tests/test_payment_processor.py::test_factory_requires_api_key PASSED [100%]

========================= 5 passed in 0.12s ==========================
```

## The Problem: Local Success, CI Failure

You commit this code and push to your repository. Your CI system runs the tests and reports:

**CI Build #47 - FAILED**

Let's examine what went wrong and why CI environments expose problems that local testing misses.

### Iteration 1: The First CI Failure

Your CI system attempts to run the tests:

```bash
# CI Log Output
Step 1/5: Checkout code
✓ Repository cloned successfully

Step 2/5: Set up Python
✓ Python 3.11.2 installed

Step 3/5: Install dependencies
ERROR: Could not find a version that satisfies the requirement requests
ERROR: No matching distribution found for requests

Build failed in 23 seconds
```

### Diagnostic Analysis: Reading the CI Failure

**The complete output**:
```
ERROR: Could not find a version that satisfies the requirement requests
ERROR: No matching distribution found for requests
```

**Let's parse this section by section**:

1. **The error type**: `ERROR: Could not find a version`
   - What this tells us: The dependency installation step failed
   - This happens before tests even run

2. **The missing component**: `requirement requests`
   - What this tells us: The `requests` library isn't being installed
   - Our code imports `requests`, so tests will fail without it

3. **Why this worked locally**:
   - On your machine: `requests` was already installed (probably for another project)
   - In CI: Fresh environment every time, nothing pre-installed

**Root cause identified**: No dependency specification file

**Why the current approach can't solve this**: CI environments start from a clean slate. They don't have your local packages.

**What we need**: A `requirements.txt` file that explicitly lists all dependencies.

### Solution: Explicit Dependency Declaration

Create a requirements file that CI can use:

```text
# requirements.txt
requests==2.31.0
pytest==7.4.3
```

Now CI can install dependencies before running tests. But we need to tell CI *how* to use this file.

### The Minimal CI Configuration

Every CI system needs instructions. Here's the conceptual structure (we'll see specific implementations in later sections):

**What CI needs to know**:
1. What programming language/runtime to use
2. How to install dependencies
3. How to run the tests
4. What constitutes success vs. failure

**Generic CI workflow**:
```
1. Provision a clean environment (container/VM)
2. Check out the code from the repository
3. Install the specified Python version
4. Install dependencies from requirements.txt
5. Run pytest
6. Report results (exit code 0 = success, non-zero = failure)
```

Let's see this in a simple shell script form to understand the mechanics:

```bash
#!/bin/bash
# ci_test.sh - What CI systems do under the hood

set -e  # Exit on any error

echo "Step 1: Environment info"
python --version
pip --version

echo "Step 2: Install dependencies"
pip install -r requirements.txt

echo "Step 3: Run tests"
pytest tests/ -v

echo "Step 4: Success!"
exit 0
```

If you run this script locally, it simulates what CI does:

```bash
$ chmod +x ci_test.sh
$ ./ci_test.sh
Step 1: Environment info
Python 3.11.2
pip 23.0.1

Step 2: Install dependencies
Collecting requests==2.31.0
  Using cached requests-2.31.0-py3-none-any.whl (62 kB)
Collecting pytest==7.4.3
  Using cached pytest-7.4.3-py3-none-any.whl (325 kB)
Installing collected packages: requests, pytest
Successfully installed pytest-7.4.3 requests-2.31.0

Step 3: Run tests
======================== test session starts =========================
collected 5 items

tests/test_payment_processor.py::test_successful_payment PASSED  [ 20%]
tests/test_payment_processor.py::test_payment_below_minimum PASSED [ 40%]
tests/test_payment_processor.py::test_gateway_error_handling PASSED [ 60%]
tests/test_payment_processor.py::test_factory_creates_processor PASSED [ 80%]
tests/test_payment_processor.py::test_factory_requires_api_key PASSED [100%]

========================= 5 passed in 0.12s ==========================

Step 4: Success!
```

### Iteration 2: Environment Variable Isolation

With dependencies fixed, CI runs again. This time, a different failure:

```bash
# CI Log Output
Step 3/5: Run tests
======================== test session starts =========================
collected 5 items

tests/test_payment_processor.py::test_successful_payment PASSED  [ 20%]
tests/test_payment_processor.py::test_payment_below_minimum PASSED [ 40%]
tests/test_payment_processor.py::test_gateway_error_handling PASSED [ 60%]
tests/test_payment_processor.py::test_factory_creates_processor FAILED [ 80%]
tests/test_payment_processor.py::test_factory_requires_api_key PASSED [100%]

============================== FAILURES ==============================
______________ test_factory_creates_processor _______________

    @patch.dict("os.environ", {"PAYMENT_API_KEY": "test_key_123"})
    def test_factory_creates_processor():
        """Test factory function reads from environment."""
        processor = create_processor_from_env()
        assert processor is not None
>       assert processor.gateway.api_key == "test_key_123"
E       AssertionError: assert 'sk_live_abc123' == 'test_key_123'
E         - test_key_123
E         + sk_live_abc123

tests/test_payment_processor.py:58: AssertionError
===================== 1 failed, 4 passed in 0.15s ====================
```

### Diagnostic Analysis: Environment Pollution

**The complete output**:
```
E       AssertionError: assert 'sk_live_abc123' == 'test_key_123'
E         - test_key_123
E         + sk_live_abc123
```

**Let's parse this section by section**:

1. **The test that failed**: `test_factory_creates_processor`
   - What this tells us: The test that uses `@patch.dict` to set environment variables

2. **The assertion introspection**:
   ```
   assert 'sk_live_abc123' == 'test_key_123'
   ```
   - What this tells us: The API key is `sk_live_abc123` instead of our test value
   - That looks like a real production API key!

3. **Why this happened**:
   - Someone set `PAYMENT_API_KEY=sk_live_abc123` in the CI environment
   - `@patch.dict` adds to the environment but doesn't clear existing values
   - The real key takes precedence over our test key

**Root cause identified**: Environment variables from CI configuration leak into tests

**Why the current approach can't solve this**: `@patch.dict` with a single key doesn't isolate the test from pre-existing environment variables.

**What we need**: Complete environment isolation for tests that depend on environment variables.

### Solution: Proper Environment Isolation

We need to ensure tests run in a clean environment state:

```python
# tests/test_payment_processor.py (updated)
import os
from decimal import Decimal
from unittest.mock import Mock, patch
import pytest
from src.payment_processor import PaymentProcessor, PaymentGateway, create_processor_from_env

@pytest.fixture
def clean_environment():
    """Ensure clean environment for tests that use environment variables."""
    # Save original environment
    original_env = os.environ.copy()
    
    # Clear payment-related variables
    for key in list(os.environ.keys()):
        if key.startswith("PAYMENT_"):
            del os.environ[key]
    
    yield
    
    # Restore original environment
    os.environ.clear()
    os.environ.update(original_env)

@pytest.fixture
def mock_gateway():
    """Create a mock payment gateway."""
    gateway = Mock(spec=PaymentGateway)
    gateway.charge.return_value = {"id": "txn_12345", "status": "succeeded"}
    return gateway

@pytest.fixture
def processor(mock_gateway):
    """Create a payment processor with mocked gateway."""
    return PaymentProcessor(mock_gateway)

def test_successful_payment(processor, mock_gateway):
    """Test processing a valid payment."""
    result = processor.process_payment(Decimal("10.00"), "card_tok_visa")
    
    assert result["success"] is True
    assert result["transaction_id"] == "txn_12345"
    assert "successfully" in result["message"]
    
    mock_gateway.charge.assert_called_once_with(Decimal("10.00"), "card_tok_visa")

def test_payment_below_minimum(processor, mock_gateway):
    """Test payment amount below minimum threshold."""
    result = processor.process_payment(Decimal("0.25"), "card_tok_visa")
    
    assert result["success"] is False
    assert result["transaction_id"] is None
    assert "at least" in result["message"]
    
    mock_gateway.charge.assert_not_called()

def test_gateway_error_handling(processor, mock_gateway):
    """Test handling of gateway API errors."""
    import requests
    mock_gateway.charge.side_effect = requests.RequestException("Network timeout")
    
    result = processor.process_payment(Decimal("10.00"), "card_tok_visa")
    
    assert result["success"] is False
    assert result["transaction_id"] is None
    assert "gateway error" in result["message"]

def test_factory_creates_processor(clean_environment):
    """Test factory function reads from environment."""
    # Now we explicitly set ONLY what we want
    os.environ["PAYMENT_API_KEY"] = "test_key_123"
    
    processor = create_processor_from_env()
    assert processor is not None
    assert processor.gateway.api_key == "test_key_123"

def test_factory_requires_api_key(clean_environment):
    """Test factory fails without API key."""
    # clean_environment ensures no PAYMENT_API_KEY exists
    with pytest.raises(ValueError, match="PAYMENT_API_KEY"):
        create_processor_from_env()

def test_factory_uses_custom_url(clean_environment):
    """Test factory respects custom API URL."""
    os.environ["PAYMENT_API_KEY"] = "test_key_123"
    os.environ["PAYMENT_API_URL"] = "https://test.example.com"
    
    processor = create_processor_from_env()
    assert processor.gateway.base_url == "https://test.example.com"
```

### Verification: Tests Now Pass in CI

With proper environment isolation:

```bash
$ pytest tests/test_payment_processor.py -v
======================== test session starts =========================
collected 6 items

tests/test_payment_processor.py::test_successful_payment PASSED  [ 16%]
tests/test_payment_processor.py::test_payment_below_minimum PASSED [ 33%]
tests/test_payment_processor.py::test_gateway_error_handling PASSED [ 50%]
tests/test_payment_processor.py::test_factory_creates_processor PASSED [ 66%]
tests/test_payment_processor.py::test_factory_requires_api_key PASSED [ 83%]
tests/test_payment_processor.py::test_factory_uses_custom_url PASSED [100%]

========================= 6 passed in 0.14s ==========================
```

## Key Principles for CI-Ready Tests

### 1. Explicit Dependencies

**Problem**: "Works on my machine" because you have packages installed globally.

**Solution**: `requirements.txt` or `pyproject.toml` with pinned versions.

### 2. Environment Isolation

**Problem**: Tests pass locally but fail in CI due to environment variable pollution.

**Solution**: Fixtures that clean and restore environment state.

### 3. No External Dependencies

**Problem**: Tests that make real HTTP requests fail when CI has no internet access or rate limits.

**Solution**: Mock external services (as we did with `mock_gateway`).

### 4. Deterministic Behavior

**Problem**: Tests that depend on current time, random values, or external state are flaky.

**Solution**: Inject dependencies, use fixed seeds, mock time-dependent functions.

### 5. Fast Execution

**Problem**: Slow tests delay feedback and waste CI resources.

**Solution**: Use mocks instead of real I/O, parallelize tests, optimize fixtures.

### The CI Mindset

When writing tests, always ask:

- **Will this work in a fresh environment?** (No pre-installed packages)
- **Will this work without my local configuration?** (No .env files, no global settings)
- **Will this work in parallel?** (No shared state between tests)
- **Will this work 100 times in a row?** (No flaky timing dependencies)

These questions guide you toward tests that are reliable in automation.

## GitHub Actions for Python Testing

## GitHub Actions: CI in Your Repository

GitHub Actions is a CI/CD platform built into GitHub. It runs workflows defined in YAML files stored in your repository. When you push code or open a pull request, GitHub automatically runs your tests.

### Why GitHub Actions?

- **Zero setup**: No external service to configure
- **Free for public repositories**: Generous free tier for private repos
- **Matrix testing**: Test across multiple Python versions simultaneously
- **Rich ecosystem**: Thousands of pre-built actions for common tasks

### The Workflow File Structure

GitHub Actions workflows live in `.github/workflows/` in your repository. Let's build one step by step.

### Iteration 1: The Minimal Workflow

Create the workflow file:

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
      
      - name: Run tests
        run: |
          pytest tests/ -v
```

### Understanding the Workflow Anatomy

Let's parse each section:

**`name: Tests`**: The workflow name shown in GitHub's UI.

**`on: [push, pull_request]`**: When to run this workflow.
- `push`: Every time code is pushed to any branch
- `pull_request`: When a PR is opened or updated

**`jobs:`**: A workflow contains one or more jobs. Jobs run in parallel by default.

**`runs-on: ubuntu-latest`**: The operating system for the job. Options:
- `ubuntu-latest`: Linux (most common for Python)
- `windows-latest`: Windows
- `macos-latest`: macOS

**`steps:`**: Sequential tasks within a job.

**`uses: actions/checkout@v4`**: A pre-built action that clones your repository.

**`uses: actions/setup-python@v4`**: Installs Python. The `with:` section specifies version.

**`run: |`**: Executes shell commands. The `|` allows multi-line scripts.

### Seeing It in Action

Commit this file and push:

```bash
$ git add .github/workflows/test.yml
$ git commit -m "Add GitHub Actions workflow"
$ git push origin main
```

GitHub automatically detects the workflow and runs it. You can watch the progress:

1. Go to your repository on GitHub
2. Click the "Actions" tab
3. See your workflow running in real-time

The output looks like this:

```text
Run actions/checkout@v4
Syncing repository: owner/repo
Fetching the repository
✓ Repository cloned

Run actions/setup-python@v4
Setup Python 3.11.2
✓ Python installed

Run pip install -r requirements.txt
Collecting requests==2.31.0
Collecting pytest==7.4.3
✓ Dependencies installed

Run pytest tests/ -v
======================== test session starts =========================
collected 6 items

tests/test_payment_processor.py::test_successful_payment PASSED  [ 16%]
tests/test_payment_processor.py::test_payment_below_minimum PASSED [ 33%]
tests/test_payment_processor.py::test_gateway_error_handling PASSED [ 50%]
tests/test_payment_processor.py::test_factory_creates_processor PASSED [ 66%]
tests/test_payment_processor.py::test_factory_requires_api_key PASSED [ 83%]
tests/test_payment_processor.py::test_factory_uses_custom_url PASSED [100%]

========================= 6 passed in 0.14s ==========================
✓ Tests passed
```

### Iteration 2: Testing Multiple Python Versions

Your users might run Python 3.9, 3.10, 3.11, or 3.12. You should test all of them. GitHub Actions makes this trivial with **matrix strategy**:

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11', '3.12']
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
      
      - name: Run tests
        run: |
          pytest tests/ -v
```

### Understanding Matrix Strategy

**`strategy: matrix:`**: Defines variables that create multiple job instances.

**`python-version: ['3.9', '3.10', '3.11', '3.12']`**: Creates 4 parallel jobs, one for each Python version.

**`${{ matrix.python-version }}`**: Template syntax that inserts the current matrix value.

When you push this change, GitHub runs **4 jobs in parallel**:

```
✓ test (3.9)  - 6 passed in 0.14s
✓ test (3.10) - 6 passed in 0.13s
✓ test (3.11) - 6 passed in 0.14s
✓ test (3.12) - 6 passed in 0.15s
```

If any version fails, you see exactly which one:

```
✓ test (3.9)  - 6 passed
✓ test (3.10) - 6 passed
✗ test (3.11) - 5 passed, 1 failed
✓ test (3.12) - 6 passed
```

### Iteration 3: Caching Dependencies

Installing dependencies on every run is slow. GitHub Actions can cache them:

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11', '3.12']
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
      
      - name: Run tests
        run: |
          pytest tests/ -v
```

**`cache: 'pip'`**: Automatically caches pip packages based on `requirements.txt` hash.

**First run**: Downloads and caches dependencies (slow).
**Subsequent runs**: Restores from cache (fast).

The output shows the cache in action:

```text
Run actions/setup-python@v4
Setup Python 3.11.2
Cache restored from key: setup-python-Linux-pip-abc123...
✓ Python installed with cached dependencies

Run pip install -r requirements.txt
Requirement already satisfied: requests==2.31.0
Requirement already satisfied: pytest==7.4.3
✓ Dependencies installed (0.3s instead of 12s)
```

### Iteration 4: Adding Code Coverage

Let's integrate coverage reporting into CI:

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11', '3.12']
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest-cov
      
      - name: Run tests with coverage
        run: |
          pytest tests/ --cov=src --cov-report=xml --cov-report=term
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          flags: unittests
          name: codecov-${{ matrix.python-version }}
```

**New additions**:

- `pip install pytest-cov`: Adds coverage plugin
- `--cov=src`: Measure coverage of the `src/` directory
- `--cov-report=xml`: Generate XML report for Codecov
- `--cov-report=term`: Show coverage in terminal output
- `codecov/codecov-action@v3`: Uploads coverage to Codecov.io (free for open source)

The output now includes coverage:

```text
Run pytest tests/ --cov=src --cov-report=xml --cov-report=term
======================== test session starts =========================
collected 6 items

tests/test_payment_processor.py::test_successful_payment PASSED  [ 16%]
tests/test_payment_processor.py::test_payment_below_minimum PASSED [ 33%]
tests/test_payment_processor.py::test_gateway_error_handling PASSED [ 50%]
tests/test_payment_processor.py::test_factory_creates_processor PASSED [ 66%]
tests/test_payment_processor.py::test_factory_requires_api_key PASSED [ 83%]
tests/test_payment_processor.py::test_factory_uses_custom_url PASSED [100%]

---------- coverage: platform linux, python 3.11.2 -----------
Name                            Stmts   Miss  Cover
---------------------------------------------------
src/payment_processor.py           45      2    96%
---------------------------------------------------
TOTAL                              45      2    96%

========================= 6 passed in 0.18s ==========================

Run codecov/codecov-action@v3
✓ Coverage uploaded to Codecov
```

### Iteration 5: Failing Fast and Providing Feedback

When tests fail, you want immediate feedback. Let's add failure handling:

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ['3.9', '3.10', '3.11', '3.12']
      fail-fast: false
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest-cov
      
      - name: Run tests with coverage
        run: |
          pytest tests/ --cov=src --cov-report=xml --cov-report=term -v
      
      - name: Upload coverage to Codecov
        if: matrix.python-version == '3.11'
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          flags: unittests
          name: codecov-umbrella
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results-${{ matrix.python-version }}
          path: |
            coverage.xml
            .coverage
```

**New additions**:

**`fail-fast: false`**: Don't cancel other matrix jobs when one fails. This lets you see which Python versions pass and which fail.

**`if: matrix.python-version == '3.11'`**: Only upload coverage once (from Python 3.11) instead of 4 times.

**`if: always()`**: Run this step even if tests fail. Useful for uploading test artifacts.

**`actions/upload-artifact@v3`**: Saves files from the workflow. You can download them later to debug failures.

### When Tests Fail in CI

Let's simulate a failure. Suppose we introduce a bug:

```python
# src/payment_processor.py (with bug)
def process_payment(self, amount: Decimal, card_token: str) -> dict:
    """Process a payment with validation."""
    if amount < self.min_amount:
        return {
            "success": False,
            "transaction_id": None,
            "message": f"Amount must be at least {self.min_amount}"
        }
    
    try:
        result = self.gateway.charge(amount, card_token)
        return {
            "success": True,
            "transaction_id": result["id"],
            "message": "Payment processed successfully"
        }
    except requests.RequestException as e:
        # BUG: Forgot to return here!
        pass
```

GitHub Actions shows the failure clearly:

```text
Run pytest tests/ --cov=src --cov-report=xml --cov-report=term -v
======================== test session starts =========================
collected 6 items

tests/test_payment_processor.py::test_successful_payment PASSED  [ 16%]
tests/test_payment_processor.py::test_payment_below_minimum PASSED [ 33%]
tests/test_payment_processor.py::test_gateway_error_handling FAILED [ 50%]

============================== FAILURES ==============================
______________ test_gateway_error_handling _______________

processor = <src.payment_processor.PaymentProcessor object at 0x7f8b3c>
mock_gateway = <Mock spec='PaymentGateway' id='140234567890'>

    def test_gateway_error_handling(processor, mock_gateway):
        """Test handling of gateway API errors."""
        import requests
        mock_gateway.charge.side_effect = requests.RequestException("Network timeout")
        
        result = processor.process_payment(Decimal("10.00"), "card_tok_visa")
        
        assert result["success"] is False
>       assert result["transaction_id"] is None
E       TypeError: 'NoneType' object is not subscriptable

tests/test_payment_processor.py:48: TypeError
===================== 1 failed, 5 passed in 0.15s ====================

Error: Process completed with exit code 1.
```

The workflow fails, and GitHub:

1. **Marks the commit with a red X** in the commit history
2. **Blocks PR merging** if you've configured branch protection
3. **Sends notifications** to the commit author
4. **Shows the exact failure** in the Actions tab

You can click through to see the full output, download artifacts, and debug.

### The Complete Production Workflow

Here's a full-featured workflow for a production project:

```yaml
# .github/workflows/test.yml
name: Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.9', '3.10', '3.11', '3.12']
        exclude:
          # Skip some combinations to save CI time
          - os: windows-latest
            python-version: '3.9'
          - os: macos-latest
            python-version: '3.9'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest-cov pytest-xdist
      
      - name: Run tests with coverage
        run: |
          pytest tests/ \
            --cov=src \
            --cov-report=xml \
            --cov-report=term \
            --cov-fail-under=80 \
            -n auto \
            -v
      
      - name: Upload coverage
        if: matrix.os == 'ubuntu-latest' && matrix.python-version == '3.11'
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          flags: unittests
          name: codecov-umbrella
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-results-${{ matrix.os }}-${{ matrix.python-version }}
          path: |
            coverage.xml
            .coverage
  
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'
      
      - name: Install linting tools
        run: |
          pip install black flake8 mypy
      
      - name: Check code formatting
        run: black --check src/ tests/
      
      - name: Lint with flake8
        run: flake8 src/ tests/ --max-line-length=100
      
      - name: Type check with mypy
        run: mypy src/ --ignore-missing-imports
```

**Advanced features**:

**Multi-OS testing**: `matrix.os` tests on Linux, Windows, and macOS.

**Selective exclusions**: `exclude:` skips specific combinations to reduce CI time.

**Coverage threshold**: `--cov-fail-under=80` fails if coverage drops below 80%.

**Parallel execution**: `-n auto` uses pytest-xdist to run tests in parallel.

**Separate lint job**: Code quality checks run independently from tests.

**Branch filtering**: Only runs on `main` and `develop` branches for pushes.

### GitHub Actions Best Practices

1. **Use caching**: Speeds up workflows significantly
2. **Fail fast = false**: See all failures, not just the first
3. **Upload artifacts**: Save test results and coverage for debugging
4. **Matrix testing**: Test multiple Python versions and OSes
5. **Separate jobs**: Linting, testing, and building should be independent
6. **Branch protection**: Require passing tests before merging PRs

## GitLab CI Integration

## GitLab CI: Pipeline-Based Testing

GitLab CI/CD uses a different model than GitHub Actions. Instead of workflows with jobs, GitLab uses **pipelines** with **stages**. Tests are defined in a `.gitlab-ci.yml` file at the repository root.

### Why GitLab CI?

- **Integrated with GitLab**: Built into GitLab's platform
- **Pipeline visualization**: Excellent UI for complex pipelines
- **Docker-first**: Native Docker support for consistent environments
- **Self-hosted runners**: Run CI on your own infrastructure

### The Pipeline Structure

GitLab CI organizes work into **stages** that run sequentially, with **jobs** within each stage running in parallel.

### Iteration 1: The Minimal Pipeline

Create the pipeline configuration:

```yaml
# .gitlab-ci.yml
image: python:3.11

stages:
  - test

test_job:
  stage: test
  script:
    - pip install -r requirements.txt
    - pytest tests/ -v
```

### Understanding the Pipeline Anatomy

**`image: python:3.11`**: The Docker image to use. GitLab runs each job in a fresh container.

**`stages:`**: Defines the pipeline stages. Jobs in the same stage run in parallel; stages run sequentially.

**`test_job:`**: A job name (can be anything).

**`stage: test`**: Assigns this job to the "test" stage.

**`script:`**: Commands to execute. Each line runs in sequence.

### How GitLab CI Differs from GitHub Actions

| Aspect              | GitHub Actions                  | GitLab CI                       |
| ------------------- | ------------------------------- | ------------------------------- |
| **Configuration**   | `.github/workflows/*.yml`       | `.gitlab-ci.yml` (root)         |
| **Execution model** | Workflows → Jobs → Steps        | Pipelines → Stages → Jobs       |
| **Environment**     | VM-based (can use containers)   | Docker-first (containers)       |
| **Parallelism**     | Matrix strategy                 | Multiple jobs in same stage     |
| **Caching**         | Explicit cache action           | Built-in `cache:` directive     |
| **Artifacts**       | Upload/download actions         | Built-in `artifacts:` directive |

### Seeing It in Action

Commit and push:

```bash
$ git add .gitlab-ci.yml
$ git commit -m "Add GitLab CI pipeline"
$ git push origin main
```

GitLab automatically detects the pipeline and runs it. Navigate to **CI/CD → Pipelines** in your GitLab project to see:

```
Pipeline #123 - passed
├─ test (stage)
   └─ test_job - passed (14s)
```

Click on `test_job` to see the output:

```text
Running with gitlab-runner 15.8.0
Preparing the "docker" executor
Using Docker executor with image python:3.11 ...
Pulling docker image python:3.11 ...
✓ Docker image pulled

Preparing environment
Running on runner-abc123...

Getting source from Git repository
Fetching changes...
✓ Repository cloned

Executing "step_script" stage of the job script
$ pip install -r requirements.txt
Collecting requests==2.31.0
Collecting pytest==7.4.3
✓ Dependencies installed

$ pytest tests/ -v
======================== test session starts =========================
collected 6 items

tests/test_payment_processor.py::test_successful_payment PASSED  [ 16%]
tests/test_payment_processor.py::test_payment_below_minimum PASSED [ 33%]
tests/test_payment_processor.py::test_gateway_error_handling PASSED [ 50%]
tests/test_payment_processor.py::test_factory_creates_processor PASSED [ 66%]
tests/test_payment_processor.py::test_factory_requires_api_key PASSED [ 83%]
tests/test_payment_processor.py::test_factory_uses_custom_url PASSED [100%]

========================= 6 passed in 0.14s ==========================

Job succeeded
```

### Iteration 2: Testing Multiple Python Versions

GitLab CI uses **parallel jobs** instead of matrix strategy. Create multiple jobs with different images:

```yaml
# .gitlab-ci.yml
stages:
  - test

.test_template: &test_template
  stage: test
  script:
    - pip install -r requirements.txt
    - pytest tests/ -v

test_python39:
  <<: *test_template
  image: python:3.9

test_python310:
  <<: *test_template
  image: python:3.10

test_python311:
  <<: *test_template
  image: python:3.11

test_python312:
  <<: *test_template
  image: python:3.12
```

### Understanding YAML Anchors

**`.test_template: &test_template`**: Defines a reusable template with an anchor (`&`).

**`<<: *test_template`**: Merges the template into this job using an alias (`*`).

This is YAML's way of avoiding repetition. It's equivalent to:

```yaml
test_python39:
  stage: test
  image: python:3.9
  script:
    - pip install -r requirements.txt
    - pytest tests/ -v
```

But more maintainable when you have many similar jobs.

### Pipeline Visualization

GitLab shows all 4 jobs running in parallel:

```
Pipeline #124 - passed
├─ test (stage)
   ├─ test_python39  - passed (12s)
   ├─ test_python310 - passed (11s)
   ├─ test_python311 - passed (13s)
   └─ test_python312 - passed (14s)
```

### Iteration 3: Caching Dependencies

Installing dependencies on every run is slow. GitLab CI has built-in caching:

```yaml
# .gitlab-ci.yml
stages:
  - test

.test_template: &test_template
  stage: test
  cache:
    key: ${CI_COMMIT_REF_SLUG}-${CI_JOB_IMAGE}
    paths:
      - .cache/pip
  before_script:
    - export PIP_CACHE_DIR="$CI_PROJECT_DIR/.cache/pip"
    - pip install -r requirements.txt
  script:
    - pytest tests/ -v

test_python39:
  <<: *test_template
  image: python:3.9

test_python310:
  <<: *test_template
  image: python:3.10

test_python311:
  <<: *test_template
  image: python:3.11

test_python312:
  <<: *test_template
  image: python:3.12
```

### Understanding GitLab Caching

**`cache:`**: Defines what to cache between pipeline runs.

**`key:`**: Cache identifier. Using `${CI_COMMIT_REF_SLUG}-${CI_JOB_IMAGE}` creates separate caches for:
- Each branch (`CI_COMMIT_REF_SLUG`)
- Each Python version (`CI_JOB_IMAGE`)

**`paths:`**: Directories to cache. We cache pip's download directory.

**`before_script:`**: Runs before the main `script:`. We configure pip to use our cached directory.

**First run**: Downloads packages and caches them (slow).
**Subsequent runs**: Restores from cache (fast).

The output shows caching in action:

```text
Restoring cache
Checking cache for main-python:3.11...
Successfully extracted cache

$ export PIP_CACHE_DIR="$CI_PROJECT_DIR/.cache/pip"
$ pip install -r requirements.txt
Requirement already satisfied: requests==2.31.0 (from cache)
Requirement already satisfied: pytest==7.4.3 (from cache)
✓ Dependencies installed (0.4s instead of 11s)
```

### Iteration 4: Adding Code Coverage

Integrate coverage reporting:

```yaml
# .gitlab-ci.yml
stages:
  - test
  - report

.test_template: &test_template
  stage: test
  cache:
    key: ${CI_COMMIT_REF_SLUG}-${CI_JOB_IMAGE}
    paths:
      - .cache/pip
  before_script:
    - export PIP_CACHE_DIR="$CI_PROJECT_DIR/.cache/pip"
    - pip install -r requirements.txt
    - pip install pytest-cov
  script:
    - pytest tests/ --cov=src --cov-report=xml --cov-report=term -v
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    paths:
      - coverage.xml
    expire_in: 1 week

test_python39:
  <<: *test_template
  image: python:3.9

test_python310:
  <<: *test_template
  image: python:3.10

test_python311:
  <<: *test_template
  image: python:3.11

test_python312:
  <<: *test_template
  image: python:3.12

coverage_report:
  stage: report
  image: python:3.11
  dependencies:
    - test_python311
  script:
    - pip install coverage
    - coverage report
    - coverage html
  artifacts:
    paths:
      - htmlcov/
    expire_in: 1 week
  coverage: '/TOTAL.*\s+(\d+%)$/'
```

### Understanding GitLab Artifacts

**`artifacts:`**: Files to save after the job completes.

**`reports: coverage_report:`**: GitLab parses this as coverage data and displays it in the UI.

**`paths:`**: Files to save. Other jobs can download these.

**`expire_in:`**: How long to keep artifacts. Options: `1 day`, `1 week`, `1 month`, `never`.

**`dependencies:`**: Which jobs' artifacts to download. `coverage_report` only needs artifacts from `test_python311`.

**`coverage: '/TOTAL.*\s+(\d+%)$/'`**: Regex to extract coverage percentage from output. GitLab displays this as a badge.

### Pipeline Visualization with Stages

Now the pipeline has two stages:

```
Pipeline #125 - passed
├─ test (stage)
│  ├─ test_python39  - passed (12s)
│  ├─ test_python310 - passed (11s)
│  ├─ test_python311 - passed (13s)
│  └─ test_python312 - passed (14s)
└─ report (stage)
   └─ coverage_report - passed (8s)
```

The `report` stage only runs if all `test` stage jobs pass.

### Iteration 5: Adding Quality Gates

Fail the pipeline if coverage drops below a threshold:

```yaml
# .gitlab-ci.yml
stages:
  - test
  - report
  - quality

.test_template: &test_template
  stage: test
  cache:
    key: ${CI_COMMIT_REF_SLUG}-${CI_JOB_IMAGE}
    paths:
      - .cache/pip
  before_script:
    - export PIP_CACHE_DIR="$CI_PROJECT_DIR/.cache/pip"
    - pip install -r requirements.txt
    - pip install pytest-cov
  script:
    - pytest tests/ --cov=src --cov-report=xml --cov-report=term --cov-fail-under=80 -v
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    paths:
      - coverage.xml
    expire_in: 1 week

test_python39:
  <<: *test_template
  image: python:3.9

test_python310:
  <<: *test_template
  image: python:3.10

test_python311:
  <<: *test_template
  image: python:3.11

test_python312:
  <<: *test_template
  image: python:3.12

coverage_report:
  stage: report
  image: python:3.11
  dependencies:
    - test_python311
  script:
    - pip install coverage
    - coverage report
    - coverage html
  artifacts:
    paths:
      - htmlcov/
    expire_in: 1 week
  coverage: '/TOTAL.*\s+(\d+%)$/'

code_quality:
  stage: quality
  image: python:3.11
  cache:
    key: ${CI_COMMIT_REF_SLUG}-quality
    paths:
      - .cache/pip
  before_script:
    - export PIP_CACHE_DIR="$CI_PROJECT_DIR/.cache/pip"
    - pip install black flake8 mypy
  script:
    - black --check src/ tests/
    - flake8 src/ tests/ --max-line-length=100
    - mypy src/ --ignore-missing-imports
  allow_failure: false
```

**New additions**:

**`--cov-fail-under=80`**: Tests fail if coverage is below 80%.

**`code_quality` job**: Runs linting and type checking.

**`allow_failure: false`**: Pipeline fails if this job fails (default behavior, but explicit here).

### When Tests Fail in GitLab CI

Let's simulate the same bug from the GitHub Actions section:

```python
# src/payment_processor.py (with bug)
def process_payment(self, amount: Decimal, card_token: str) -> dict:
    """Process a payment with validation."""
    if amount < self.min_amount:
        return {
            "success": False,
            "transaction_id": None,
            "message": f"Amount must be at least {self.min_amount}"
        }
    
    try:
        result = self.gateway.charge(amount, card_token)
        return {
            "success": True,
            "transaction_id": result["id"],
            "message": "Payment processed successfully"
        }
    except requests.RequestException as e:
        # BUG: Forgot to return here!
        pass
```

GitLab shows the failure:

```text
$ pytest tests/ --cov=src --cov-report=xml --cov-report=term --cov-fail-under=80 -v
======================== test session starts =========================
collected 6 items

tests/test_payment_processor.py::test_successful_payment PASSED  [ 16%]
tests/test_payment_processor.py::test_payment_below_minimum PASSED [ 33%]
tests/test_payment_processor.py::test_gateway_error_handling FAILED [ 50%]

============================== FAILURES ==============================
______________ test_gateway_error_handling _______________

processor = <src.payment_processor.PaymentProcessor object at 0x7f8b3c>
mock_gateway = <Mock spec='PaymentGateway' id='140234567890'>

    def test_gateway_error_handling(processor, mock_gateway):
        """Test handling of gateway API errors."""
        import requests
        mock_gateway.charge.side_effect = requests.RequestException("Network timeout")
        
        result = processor.process_payment(Decimal("10.00"), "card_tok_visa")
        
        assert result["success"] is False
>       assert result["transaction_id"] is None
E       TypeError: 'NoneType' object is not subscriptable

tests/test_payment_processor.py:48: TypeError
===================== 1 failed, 5 passed in 0.15s ====================

ERROR: Job failed: exit code 1
```

The pipeline visualization shows:

```
Pipeline #126 - failed
├─ test (stage)
│  ├─ test_python39  - passed (12s)
│  ├─ test_python310 - passed (11s)
│  ├─ test_python311 - failed (13s) ← Failure here
│  └─ test_python312 - passed (14s)
└─ report (stage) - skipped (previous stage failed)
```

GitLab:

1. **Marks the commit with a red X**
2. **Blocks merge requests** if you've configured merge request approvals
3. **Sends notifications** via email or Slack
4. **Shows the failure** in the pipeline view

You can click the failed job to see full output and download artifacts.

### The Complete Production Pipeline

Here's a full-featured pipeline for a production project:

```yaml
# .gitlab-ci.yml
variables:
  PIP_CACHE_DIR: "$CI_PROJECT_DIR/.cache/pip"

stages:
  - build
  - test
  - report
  - quality
  - deploy

cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .cache/pip
    - venv/

before_script:
  - python -V
  - pip install virtualenv
  - virtualenv venv
  - source venv/bin/activate

.test_template: &test_template
  stage: test
  script:
    - pip install -r requirements.txt
    - pip install pytest-cov pytest-xdist
    - pytest tests/ --cov=src --cov-report=xml --cov-report=term --cov-fail-under=80 -n auto -v
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
      junit: report.xml
    paths:
      - coverage.xml
      - report.xml
    expire_in: 1 week
  coverage: '/TOTAL.*\s+(\d+%)$/'

test_python39:
  <<: *test_template
  image: python:3.9

test_python310:
  <<: *test_template
  image: python:3.10

test_python311:
  <<: *test_template
  image: python:3.11

test_python312:
  <<: *test_template
  image: python:3.12

coverage_report:
  stage: report
  image: python:3.11
  dependencies:
    - test_python311
  script:
    - pip install coverage
    - coverage report
    - coverage html
  artifacts:
    paths:
      - htmlcov/
    expire_in: 1 month

code_quality:
  stage: quality
  image: python:3.11
  script:
    - pip install black flake8 mypy
    - black --check src/ tests/
    - flake8 src/ tests/ --max-line-length=100 --count --statistics
    - mypy src/ --ignore-missing-imports
  allow_failure: false

security_scan:
  stage: quality
  image: python:3.11
  script:
    - pip install bandit safety
    - bandit -r src/ -f json -o bandit-report.json
    - safety check --json > safety-report.json
  artifacts:
    paths:
      - bandit-report.json
      - safety-report.json
    expire_in: 1 month
  allow_failure: true

deploy_docs:
  stage: deploy
  image: python:3.11
  script:
    - pip install sphinx
    - cd docs && make html
  artifacts:
    paths:
      - docs/_build/html
  only:
    - main
```

**Advanced features**:

**Global cache**: `cache:` at the top level applies to all jobs.

**Virtual environment caching**: Cache `venv/` to avoid reinstalling packages.

**JUnit reports**: `junit: report.xml` integrates test results into GitLab's UI.

**Security scanning**: `bandit` and `safety` check for security issues.

**Conditional deployment**: `only: - main` runs `deploy_docs` only on the main branch.

**Parallel testing**: `-n auto` uses pytest-xdist for parallel execution.

### GitLab CI Best Practices

1. **Use Docker images**: Ensures consistent environments
2. **Cache aggressively**: Cache pip, venv, and build artifacts
3. **Use YAML anchors**: Reduce duplication in configuration
4. **Separate stages**: Build → Test → Report → Quality → Deploy
5. **Artifacts for debugging**: Save test results and coverage reports
6. **Merge request pipelines**: Configure pipelines to run on MRs
7. **Protected branches**: Require passing pipelines before merging

## Jenkins and Other CI Systems

## Jenkins: The Self-Hosted Powerhouse

Jenkins is the oldest and most flexible CI system. Unlike GitHub Actions and GitLab CI (which are cloud-hosted), Jenkins typically runs on your own infrastructure. This gives you complete control but requires more setup.

### Why Jenkins?

- **Self-hosted**: Run on your own servers, no external dependencies
- **Highly customizable**: Thousands of plugins for every use case
- **Enterprise features**: Advanced security, audit logs, distributed builds
- **Language-agnostic**: Not Python-specific, works with any language

### Jenkins Pipeline Structure

Jenkins uses **Jenkinsfiles** written in Groovy (a JVM language). There are two syntaxes:

1. **Declarative Pipeline**: Structured, easier to learn (we'll use this)
2. **Scripted Pipeline**: More flexible, harder to learn

### Iteration 1: The Minimal Jenkinsfile

Create a pipeline definition:

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Install Dependencies') {
            steps {
                sh 'pip install -r requirements.txt'
            }
        }
        
        stage('Run Tests') {
            steps {
                sh 'pytest tests/ -v'
            }
        }
    }
}
```

### Understanding the Jenkinsfile Anatomy

**`pipeline {}`**: The root block for declarative pipelines.

**`agent any`**: Run on any available Jenkins agent (worker node).

**`stages {}`**: Contains all pipeline stages.

**`stage('Name') {}`**: A named stage in the pipeline.

**`steps {}`**: Commands to execute in this stage.

**`checkout scm`**: Jenkins-specific command to clone the repository.

**`sh 'command'`**: Execute a shell command.

### How Jenkins Differs from GitHub Actions and GitLab CI

| Aspect              | GitHub Actions          | GitLab CI               | Jenkins                 |
| ------------------- | ----------------------- | ----------------------- | ----------------------- |
| **Hosting**         | Cloud (GitHub)          | Cloud (GitLab)          | Self-hosted             |
| **Configuration**   | YAML                    | YAML                    | Groovy (Jenkinsfile)    |
| **Execution model** | Workflows → Jobs        | Pipelines → Stages      | Pipelines → Stages      |
| **Environment**     | VM or container         | Docker containers       | Configurable (any)      |
| **Plugins**         | Actions marketplace     | Built-in + extensions   | 1800+ plugins           |
| **Setup**           | Zero (built-in)         | Zero (built-in)         | Manual installation     |

### Setting Up Jenkins (Brief Overview)

Since Jenkins is self-hosted, you need to:

1. **Install Jenkins**: Download from jenkins.io or use Docker
2. **Install plugins**: "Pipeline", "Git", "Python" plugins
3. **Configure agents**: Set up worker nodes with Python installed
4. **Create a job**: Point it to your repository

For this chapter, we'll assume Jenkins is already set up and focus on the Jenkinsfile.

### Iteration 2: Testing Multiple Python Versions

Jenkins doesn't have built-in matrix support like GitHub Actions. You need to use the `matrix` directive (added in Jenkins 2.22):

```groovy
// Jenkinsfile
pipeline {
    agent none
    
    stages {
        stage('Test') {
            matrix {
                agent any
                axes {
                    axis {
                        name 'PYTHON_VERSION'
                        values '3.9', '3.10', '3.11', '3.12'
                    }
                }
                stages {
                    stage('Setup') {
                        steps {
                            sh """
                                python${PYTHON_VERSION} -m venv venv
                                . venv/bin/activate
                                pip install -r requirements.txt
                            """
                        }
                    }
                    stage('Test') {
                        steps {
                            sh """
                                . venv/bin/activate
                                pytest tests/ -v
                            """
                        }
                    }
                }
            }
        }
    }
}
```

### Understanding Jenkins Matrix

**`agent none`**: Don't allocate an agent at the pipeline level (each matrix cell gets its own).

**`matrix {}`**: Defines a matrix build.

**`axes {}`**: Defines matrix dimensions.

**`axis { name 'VAR' values '1', '2' }`**: Creates a variable with multiple values.

**`stages {}`**: Stages to run for each matrix cell.

**`${PYTHON_VERSION}`**: Groovy variable interpolation.

**Triple-quoted strings (`"""`)**: Allow multi-line shell commands.

### Iteration 3: Adding Code Coverage

Integrate coverage with Jenkins' built-in reporting:

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt
                    pip install pytest-cov
                '''
            }
        }
        
        stage('Test') {
            steps {
                sh '''
                    . venv/bin/activate
                    pytest tests/ \
                        --cov=src \
                        --cov-report=xml \
                        --cov-report=html \
                        --cov-report=term \
                        --junitxml=report.xml \
                        -v
                '''
            }
        }
    }
    
    post {
        always {
            junit 'report.xml'
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'htmlcov',
                reportFiles: 'index.html',
                reportName: 'Coverage Report'
            ])
        }
    }
}
```

### Understanding Jenkins Post Actions

**`post {}`**: Actions to run after stages complete.

**`always {}`**: Run regardless of success or failure. Other options:
- `success {}`: Only on success
- `failure {}`: Only on failure
- `unstable {}`: When tests fail but build succeeds

**`junit 'report.xml'`**: Parse JUnit XML and display test results in Jenkins UI.

**`publishHTML([...])`**: Publish HTML reports (requires HTML Publisher plugin).

### Jenkins UI Integration

After running, Jenkins displays:

1. **Test Results**: Clickable test names, failure details, trends over time
2. **Coverage Report**: Interactive HTML coverage report
3. **Build History**: Graph showing test pass/fail trends
4. **Console Output**: Full log of the build

### Iteration 4: Parallel Stages

Jenkins can run stages in parallel for faster builds:

```groovy
// Jenkinsfile
pipeline {
    agent any
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }
        
        stage('Setup') {
            steps {
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt
                    pip install pytest-cov pytest-xdist black flake8 mypy
                '''
            }
        }
        
        stage('Parallel Checks') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh '''
                            . venv/bin/activate
                            pytest tests/ \
                                --cov=src \
                                --cov-report=xml \
                                --cov-report=html \
                                --junitxml=report.xml \
                                -n auto \
                                -v
                        '''
                    }
                }
                
                stage('Code Quality') {
                    steps {
                        sh '''
                            . venv/bin/activate
                            black --check src/ tests/
                            flake8 src/ tests/ --max-line-length=100
                            mypy src/ --ignore-missing-imports
                        '''
                    }
                }
            }
        }
    }
    
    post {
        always {
            junit 'report.xml'
            publishHTML([
                allowMissing: false,
                alwaysLinkToLastBuild: true,
                keepAll: true,
                reportDir: 'htmlcov',
                reportFiles: 'index.html',
                reportName: 'Coverage Report'
            ])
        }
    }
}
```

### Understanding Parallel Stages

**`parallel {}`**: Runs contained stages simultaneously.

**Benefits**:
- Tests and linting run at the same time
- Faster feedback (total time = max(test_time, lint_time) instead of sum)
- Better resource utilization

**Visualization in Jenkins**:
```
Pipeline
├─ Checkout (sequential)
├─ Setup (sequential)
├─ Parallel Checks
│  ├─ Unit Tests (parallel)
│  └─ Code Quality (parallel)
└─ Post Actions (sequential)
```

## Other CI Systems

### CircleCI

CircleCI is similar to GitHub Actions but with a different configuration syntax:

```yaml
# .circleci/config.yml
version: 2.1

orbs:
  python: circleci/python@2.1.1

jobs:
  test:
    docker:
      - image: cimg/python:3.11
    steps:
      - checkout
      - python/install-packages:
          pkg-manager: pip
      - run:
          name: Run tests
          command: pytest tests/ -v

workflows:
  test-workflow:
    jobs:
      - test
```

**Key features**:
- **Orbs**: Reusable configuration packages (like `python: circleci/python@2.1.1`)
- **Docker-first**: Native Docker support
- **Workflows**: Orchestrate multiple jobs
- **Caching**: Automatic dependency caching

### Travis CI

Travis CI uses a `.travis.yml` file:

```yaml
# .travis.yml
language: python

python:
  - "3.9"
  - "3.10"
  - "3.11"
  - "3.12"

install:
  - pip install -r requirements.txt

script:
  - pytest tests/ -v

after_success:
  - pip install codecov
  - codecov
```

**Key features**:
- **Simple configuration**: Very concise for basic use cases
- **Matrix builds**: Built-in support for multiple Python versions
- **Free for open source**: Popular in the open-source community

### Azure Pipelines

Azure Pipelines uses `azure-pipelines.yml`:

```yaml
# azure-pipelines.yml
trigger:
  - main

pool:
  vmImage: 'ubuntu-latest'

strategy:
  matrix:
    Python39:
      python.version: '3.9'
    Python310:
      python.version: '3.10'
    Python311:
      python.version: '3.11'
    Python312:
      python.version: '3.12'

steps:
  - task: UsePythonVersion@0
    inputs:
      versionSpec: '$(python.version)'
    displayName: 'Use Python $(python.version)'

  - script: |
      pip install -r requirements.txt
    displayName: 'Install dependencies'

  - script: |
      pytest tests/ -v
    displayName: 'Run tests'
```

**Key features**:
- **Microsoft ecosystem**: Integrates with Azure services
- **Matrix strategy**: Similar to GitHub Actions
- **Tasks**: Pre-built actions for common operations

## Choosing a CI System

### Decision Framework

| Factor                  | GitHub Actions | GitLab CI | Jenkins | CircleCI | Travis CI | Azure Pipelines |
| ----------------------- | -------------- | --------- | ------- | -------- | --------- | --------------- |
| **Setup complexity**    | Zero           | Zero      | High    | Low      | Low       | Low             |
| **Cost (open source)**  | Free           | Free      | Free    | Free     | Free      | Free            |
| **Cost (private)**      | Generous free  | Generous  | Free    | Limited  | Paid      | Generous        |
| **Self-hosting**        | No             | Yes       | Yes     | No       | No        | No              |
| **Configuration**       | YAML           | YAML      | Groovy  | YAML     | YAML      | YAML            |
| **Matrix testing**      | Excellent      | Good      | Good    | Good     | Excellent | Excellent       |
| **Docker support**      | Good           | Excellent | Good    | Excellent| Good      | Good            |
| **Learning curve**      | Low            | Low       | High    | Low      | Low       | Medium          |

### When to Choose Each

**GitHub Actions**: You're already on GitHub and want zero setup.

**GitLab CI**: You're on GitLab or need self-hosted CI with excellent Docker support.

**Jenkins**: You need complete control, have complex requirements, or must run on-premises.

**CircleCI**: You want powerful features with minimal configuration.

**Travis CI**: You're building open-source projects and want simplicity.

**Azure Pipelines**: You're in the Microsoft ecosystem or need Azure integration.

### Universal Best Practices

Regardless of which CI system you choose:

1. **Pin dependencies**: Use exact versions in `requirements.txt`
2. **Test multiple Python versions**: Don't assume your version is the only one
3. **Cache dependencies**: Speed up builds significantly
4. **Fail fast**: Stop on first failure to save time
5. **Parallel execution**: Run tests and linting simultaneously
6. **Artifacts**: Save test results and coverage reports
7. **Clear naming**: Use descriptive job/stage names
8. **Documentation**: Comment complex pipeline logic

## Generating Test Reports

## The Problem: Test Output Disappears

When tests run in CI, the output scrolls by in the console log. If you have 500 tests, finding the one that failed is painful. You need **persistent, structured test reports** that you can browse, search, and share.

### What Makes a Good Test Report?

A good test report should:

1. **Persist**: Available after the CI run completes
2. **Be browsable**: HTML format with navigation
3. **Show details**: Full failure messages, tracebacks, and context
4. **Track trends**: Compare results across runs
5. **Be shareable**: Send a link to teammates

### The Reference Implementation: Enhanced Payment Processor

Let's add more tests to our payment processor to generate meaningful reports:

```python
# tests/test_payment_processor_extended.py
from decimal import Decimal
from unittest.mock import Mock
import pytest
from src.payment_processor import PaymentProcessor, PaymentGateway

@pytest.fixture
def mock_gateway():
    """Create a mock payment gateway."""
    gateway = Mock(spec=PaymentGateway)
    gateway.charge.return_value = {"id": "txn_12345", "status": "succeeded"}
    return gateway

@pytest.fixture
def processor(mock_gateway):
    """Create a payment processor with mocked gateway."""
    return PaymentProcessor(mock_gateway)

class TestPaymentValidation:
    """Test payment amount validation."""
    
    def test_minimum_amount_accepted(self, processor):
        """Test that minimum amount is accepted."""
        result = processor.process_payment(Decimal("0.50"), "card_tok_visa")
        assert result["success"] is True
    
    def test_below_minimum_rejected(self, processor):
        """Test that below minimum is rejected."""
        result = processor.process_payment(Decimal("0.49"), "card_tok_visa")
        assert result["success"] is False
    
    def test_zero_amount_rejected(self, processor):
        """Test that zero amount is rejected."""
        result = processor.process_payment(Decimal("0.00"), "card_tok_visa")
        assert result["success"] is False
    
    def test_negative_amount_rejected(self, processor):
        """Test that negative amount is rejected."""
        result = processor.process_payment(Decimal("-10.00"), "card_tok_visa")
        assert result["success"] is False

class TestPaymentProcessing:
    """Test successful payment processing."""
    
    def test_small_payment(self, processor, mock_gateway):
        """Test processing a small payment."""
        result = processor.process_payment(Decimal("1.00"), "card_tok_visa")
        assert result["success"] is True
        assert result["transaction_id"] == "txn_12345"
    
    def test_large_payment(self, processor, mock_gateway):
        """Test processing a large payment."""
        result = processor.process_payment(Decimal("9999.99"), "card_tok_visa")
        assert result["success"] is True
    
    def test_gateway_receives_correct_amount(self, processor, mock_gateway):
        """Test that gateway receives the exact amount."""
        processor.process_payment(Decimal("42.50"), "card_tok_visa")
        mock_gateway.charge.assert_called_once_with(Decimal("42.50"), "card_tok_visa")

class TestErrorHandling:
    """Test error handling scenarios."""
    
    def test_network_timeout(self, processor, mock_gateway):
        """Test handling of network timeout."""
        import requests
        mock_gateway.charge.side_effect = requests.Timeout("Connection timeout")
        
        result = processor.process_payment(Decimal("10.00"), "card_tok_visa")
        assert result["success"] is False
        assert "gateway error" in result["message"]
    
    def test_connection_error(self, processor, mock_gateway):
        """Test handling of connection error."""
        import requests
        mock_gateway.charge.side_effect = requests.ConnectionError("Cannot connect")
        
        result = processor.process_payment(Decimal("10.00"), "card_tok_visa")
        assert result["success"] is False
    
    def test_http_error(self, processor, mock_gateway):
        """Test handling of HTTP error."""
        import requests
        mock_gateway.charge.side_effect = requests.HTTPError("500 Server Error")
        
        result = processor.process_payment(Decimal("10.00"), "card_tok_visa")
        assert result["success"] is False

@pytest.mark.parametrize("amount,expected", [
    (Decimal("0.50"), True),
    (Decimal("1.00"), True),
    (Decimal("100.00"), True),
    (Decimal("0.49"), False),
    (Decimal("0.00"), False),
])
def test_amount_validation_parametrized(processor, amount, expected):
    """Test amount validation with multiple values."""
    result = processor.process_payment(amount, "card_tok_visa")
    assert result["success"] == expected
```

Now we have 15 tests organized into classes. Let's generate reports for them.

## Iteration 1: JUnit XML Reports

JUnit XML is the universal format for test results. Every CI system understands it.

### Generating JUnit XML

Pytest has built-in JUnit XML support:

```bash
$ pytest tests/ --junitxml=report.xml -v
======================== test session starts =========================
collected 15 items

tests/test_payment_processor_extended.py::TestPaymentValidation::test_minimum_amount_accepted PASSED [  6%]
tests/test_payment_processor_extended.py::TestPaymentValidation::test_below_minimum_rejected PASSED [ 13%]
tests/test_payment_processor_extended.py::TestPaymentValidation::test_zero_amount_rejected PASSED [ 20%]
tests/test_payment_processor_extended.py::TestPaymentValidation::test_negative_amount_rejected PASSED [ 26%]
tests/test_payment_processor_extended.py::TestPaymentProcessing::test_small_payment PASSED [ 33%]
tests/test_payment_processor_extended.py::TestPaymentProcessing::test_large_payment PASSED [ 40%]
tests/test_payment_processor_extended.py::TestPaymentProcessing::test_gateway_receives_correct_amount PASSED [ 46%]
tests/test_payment_processor_extended.py::TestErrorHandling::test_network_timeout PASSED [ 53%]
tests/test_payment_processor_extended.py::TestErrorHandling::test_connection_error PASSED [ 60%]
tests/test_payment_processor_extended.py::TestErrorHandling::test_http_error PASSED [ 66%]
tests/test_payment_processor_extended.py::test_amount_validation_parametrized[0.50-True] PASSED [ 73%]
tests/test_payment_processor_extended.py::test_amount_validation_parametrized[1.00-True] PASSED [ 80%]
tests/test_payment_processor_extended.py::test_amount_validation_parametrized[100.00-True] PASSED [ 86%]
tests/test_payment_processor_extended.py::test_amount_validation_parametrized[0.49-False] PASSED [ 93%]
tests/test_payment_processor_extended.py::test_amount_validation_parametrized[0.00-False] PASSED [100%]

========================= 15 passed in 0.23s =========================
```

### Understanding the JUnit XML Format

The generated `report.xml` looks like this:

```xml
<?xml version="1.0" encoding="utf-8"?>
<testsuites>
  <testsuite name="pytest" errors="0" failures="0" skipped="0" tests="15" time="0.234">
    <testcase classname="tests.test_payment_processor_extended.TestPaymentValidation" 
              name="test_minimum_amount_accepted" 
              time="0.012" />
    <testcase classname="tests.test_payment_processor_extended.TestPaymentValidation" 
              name="test_below_minimum_rejected" 
              time="0.008" />
    <!-- More test cases... -->
    <testcase classname="tests.test_payment_processor_extended" 
              name="test_amount_validation_parametrized[0.50-True]" 
              time="0.011" />
  </testsuite>
</testsuites>
```

**Key elements**:

- `<testsuites>`: Root element
- `<testsuite>`: A collection of tests (usually one per file)
- `<testcase>`: Individual test with name, class, and execution time
- Attributes: `errors`, `failures`, `skipped`, `tests`, `time`

### When Tests Fail

Let's introduce a failure:

```python
# Temporarily break a test
def test_small_payment(self, processor, mock_gateway):
    """Test processing a small payment."""
    result = processor.process_payment(Decimal("1.00"), "card_tok_visa")
    assert result["success"] is True
    assert result["transaction_id"] == "WRONG_ID"  # This will fail
```

```bash
$ pytest tests/ --junitxml=report.xml -v
======================== test session starts =========================
collected 15 items

tests/test_payment_processor_extended.py::TestPaymentValidation::test_minimum_amount_accepted PASSED [  6%]
tests/test_payment_processor_extended.py::TestPaymentValidation::test_below_minimum_rejected PASSED [ 13%]
tests/test_payment_processor_extended.py::TestPaymentValidation::test_zero_amount_rejected PASSED [ 20%]
tests/test_payment_processor_extended.py::TestPaymentValidation::test_negative_amount_rejected PASSED [ 26%]
tests/test_payment_processor_extended.py::TestPaymentProcessing::test_small_payment FAILED [ 33%]

============================== FAILURES ==============================
______________ TestPaymentProcessing.test_small_payment ______________

self = <tests.test_payment_processor_extended.TestPaymentProcessing object at 0x7f8b3c>
processor = <src.payment_processor.PaymentProcessor object at 0x7f8b4d>
mock_gateway = <Mock spec='PaymentGateway' id='140234567890'>

    def test_small_payment(self, processor, mock_gateway):
        """Test processing a small payment."""
        result = processor.process_payment(Decimal("1.00"), "card_tok_visa")
        assert result["success"] is True
>       assert result["transaction_id"] == "WRONG_ID"
E       AssertionError: assert 'txn_12345' == 'WRONG_ID'
E         - WRONG_ID
E         + txn_12345

tests/test_payment_processor_extended.py:58: AssertionError
===================== 1 failed, 14 passed in 0.25s ====================
```

The JUnit XML now includes the failure:

```xml
<testcase classname="tests.test_payment_processor_extended.TestPaymentProcessing" 
          name="test_small_payment" 
          time="0.015">
  <failure message="AssertionError: assert 'txn_12345' == 'WRONG_ID'">
    <![CDATA[
self = <tests.test_payment_processor_extended.TestPaymentProcessing object at 0x7f8b3c>
processor = <src.payment_processor.PaymentProcessor object at 0x7f8b4d>
mock_gateway = <Mock spec='PaymentGateway' id='140234567890'>

    def test_small_payment(self, processor, mock_gateway):
        """Test processing a small payment."""
        result = processor.process_payment(Decimal("1.00"), "card_tok_visa")
        assert result["success"] is True
>       assert result["transaction_id"] == "WRONG_ID"
E       AssertionError: assert 'txn_12345' == 'WRONG_ID'
E         - WRONG_ID
E         + txn_12345

tests/test_payment_processor_extended.py:58: AssertionError
    ]]>
  </failure>
</testcase>
```

CI systems parse this XML and display it in their UI with:
- Test names
- Pass/fail status
- Execution time
- Full failure messages
- Trends over time

## Iteration 2: HTML Reports with pytest-html

JUnit XML is machine-readable but not human-friendly. HTML reports are much better for browsing.

### Installing pytest-html

```bash
$ pip install pytest-html
```

### Generating HTML Reports

```bash
$ pytest tests/ --html=report.html --self-contained-html -v
======================== test session starts =========================
collected 15 items

tests/test_payment_processor_extended.py::TestPaymentValidation::test_minimum_amount_accepted PASSED [  6%]
tests/test_payment_processor_extended.py::TestPaymentValidation::test_below_minimum_rejected PASSED [ 13%]
[... all tests ...]
========================= 15 passed in 0.23s =========================

Generated html report: file:///path/to/report.html
```

**`--html=report.html`**: Generate HTML report.

**`--self-contained-html`**: Embed CSS/JS in the HTML file (single file, easy to share).

### What the HTML Report Contains

Open `report.html` in a browser to see:

**Summary section**:
- Total tests: 15
- Passed: 15
- Failed: 0
- Skipped: 0
- Errors: 0
- Duration: 0.23s

**Test results table**:
| Test                                                                 | Result | Duration |
| -------------------------------------------------------------------- | ------ | -------- |
| tests/...::TestPaymentValidation::test_minimum_amount_accepted       | Passed | 0.012s   |
| tests/...::TestPaymentValidation::test_below_minimum_rejected        | Passed | 0.008s   |
| tests/...::TestPaymentProcessing::test_small_payment                 | Passed | 0.015s   |
| tests/...::test_amount_validation_parametrized[0.50-True]            | Passed | 0.011s   |

**Interactive features**:
- Click test names to expand details
- Filter by status (passed/failed/skipped)
- Sort by duration
- Search test names

### When Tests Fail

With the same failure from before, the HTML report shows:

```html
<!-- Simplified HTML structure -->
<tr class="failed">
  <td>tests/...::TestPaymentProcessing::test_small_payment</td>
  <td class="col-result">Failed</td>
  <td class="col-duration">0.015s</td>
</tr>
<tr class="extra">
  <td colspan="3">
    <div class="log">
      <pre>
self = &lt;tests.test_payment_processor_extended.TestPaymentProcessing object at 0x7f8b3c&gt;
processor = &lt;src.payment_processor.PaymentProcessor object at 0x7f8b4d&gt;
mock_gateway = &lt;Mock spec='PaymentGateway' id='140234567890'&gt;

    def test_small_payment(self, processor, mock_gateway):
        """Test processing a small payment."""
        result = processor.process_payment(Decimal("1.00"), "card_tok_visa")
        assert result["success"] is True
&gt;       assert result["transaction_id"] == "WRONG_ID"
E       AssertionError: assert 'txn_12345' == 'WRONG_ID'
E         - WRONG_ID
E         + txn_12345

tests/test_payment_processor_extended.py:58: AssertionError
      </pre>
    </div>
  </td>
</tr>
```

The failure is highlighted in red, and clicking it expands the full traceback.

## Iteration 3: Combining Reports with Coverage

Generate both test results and coverage in one command:

```bash
$ pytest tests/ \
    --html=report.html \
    --self-contained-html \
    --cov=src \
    --cov-report=html \
    --cov-report=term \
    --junitxml=junit.xml \
    -v
======================== test session starts =========================
collected 15 items

tests/test_payment_processor_extended.py::TestPaymentValidation::test_minimum_amount_accepted PASSED [  6%]
[... all tests ...]
========================= 15 passed in 0.23s =========================

---------- coverage: platform linux, python 3.11.2 -----------
Name                            Stmts   Miss  Cover
---------------------------------------------------
src/payment_processor.py           45      2    96%
---------------------------------------------------
TOTAL                              45      2    96%

Generated html report: file:///path/to/report.html
Generated coverage html report: file:///path/to/htmlcov/index.html
```

Now you have:
- `report.html`: Test results
- `htmlcov/index.html`: Coverage report
- `junit.xml`: Machine-readable test results

### Integrating with CI

In your CI configuration, generate reports and save them as artifacts:

```yaml
# .github/workflows/test.yml (excerpt)
- name: Run tests with reports
  run: |
    pytest tests/ \
      --html=report.html \
      --self-contained-html \
      --cov=src \
      --cov-report=html \
      --junitxml=junit.xml \
      -v

- name: Upload test results
  if: always()
  uses: actions/upload-artifact@v3
  with:
    name: test-reports
    path: |
      report.html
      junit.xml
      htmlcov/
```

After the CI run, you can:
1. Download the artifacts from the GitHub Actions UI
2. Open `report.html` locally
3. Browse `htmlcov/index.html` for coverage details

## Iteration 4: Advanced Reporting with Allure

Allure is a powerful reporting framework with rich features:

- **Test history**: Compare results across runs
- **Categorization**: Group tests by feature, severity, etc.
- **Attachments**: Screenshots, logs, API responses
- **Trends**: Graphs showing test stability over time

### Installing Allure

```bash
$ pip install allure-pytest
```

### Generating Allure Reports

```bash
$ pytest tests/ --alluredir=allure-results -v
======================== test session starts =========================
collected 15 items

tests/test_payment_processor_extended.py::TestPaymentValidation::test_minimum_amount_accepted PASSED [  6%]
[... all tests ...]
========================= 15 passed in 0.23s =========================
```

This creates `allure-results/` with JSON files. To view the report, you need the Allure command-line tool:

```bash
# Install Allure (requires Java)
$ brew install allure  # macOS
$ sudo apt-get install allure  # Ubuntu

# Generate and open the report
$ allure serve allure-results
Generating report...
Report successfully generated to /tmp/allure-report
Starting web server...
Server started at http://localhost:8080
```

### Enhancing Tests with Allure Decorators

Allure provides decorators to add metadata:

```python
# tests/test_payment_processor_allure.py
import allure
from decimal import Decimal
from unittest.mock import Mock
import pytest
from src.payment_processor import PaymentProcessor, PaymentGateway

@pytest.fixture
def mock_gateway():
    """Create a mock payment gateway."""
    gateway = Mock(spec=PaymentGateway)
    gateway.charge.return_value = {"id": "txn_12345", "status": "succeeded"}
    return gateway

@pytest.fixture
def processor(mock_gateway):
    """Create a payment processor with mocked gateway."""
    return PaymentProcessor(mock_gateway)

@allure.feature("Payment Validation")
@allure.story("Amount Validation")
class TestPaymentValidation:
    """Test payment amount validation."""
    
    @allure.title("Minimum amount should be accepted")
    @allure.severity(allure.severity_level.CRITICAL)
    def test_minimum_amount_accepted(self, processor):
        """Test that minimum amount is accepted."""
        with allure.step("Process payment with minimum amount"):
            result = processor.process_payment(Decimal("0.50"), "card_tok_visa")
        
        with allure.step("Verify payment succeeded"):
            assert result["success"] is True
    
    @allure.title("Below minimum amount should be rejected")
    @allure.severity(allure.severity_level.CRITICAL)
    def test_below_minimum_rejected(self, processor):
        """Test that below minimum is rejected."""
        with allure.step("Process payment below minimum"):
            result = processor.process_payment(Decimal("0.49"), "card_tok_visa")
        
        with allure.step("Verify payment was rejected"):
            assert result["success"] is False
            assert "at least" in result["message"]

@allure.feature("Payment Processing")
@allure.story("Successful Payments")
class TestPaymentProcessing:
    """Test successful payment processing."""
    
    @allure.title("Small payment should be processed successfully")
    @allure.severity(allure.severity_level.NORMAL)
    def test_small_payment(self, processor, mock_gateway):
        """Test processing a small payment."""
        with allure.step("Process $1.00 payment"):
            result = processor.process_payment(Decimal("1.00"), "card_tok_visa")
        
        with allure.step("Verify transaction ID returned"):
            assert result["success"] is True
            assert result["transaction_id"] == "txn_12345"
        
        with allure.step("Verify gateway was called correctly"):
            mock_gateway.charge.assert_called_once_with(Decimal("1.00"), "card_tok_visa")

@allure.feature("Error Handling")
@allure.story("Network Errors")
class TestErrorHandling:
    """Test error handling scenarios."""
    
    @allure.title("Network timeout should be handled gracefully")
    @allure.severity(allure.severity_level.CRITICAL)
    def test_network_timeout(self, processor, mock_gateway):
        """Test handling of network timeout."""
        import requests
        
        with allure.step("Configure gateway to timeout"):
            mock_gateway.charge.side_effect = requests.Timeout("Connection timeout")
        
        with allure.step("Attempt payment"):
            result = processor.process_payment(Decimal("10.00"), "card_tok_visa")
        
        with allure.step("Verify error was handled"):
            assert result["success"] is False
            assert "gateway error" in result["message"]
            allure.attach(result["message"], name="Error Message", attachment_type=allure.attachment_type.TEXT)
```

### Understanding Allure Decorators

**`@allure.feature("Name")`**: Groups tests by feature (e.g., "Payment Validation").

**`@allure.story("Name")`**: Sub-groups within a feature (e.g., "Amount Validation").

**`@allure.title("Description")`**: Human-readable test name.

**`@allure.severity(level)`**: Priority level (BLOCKER, CRITICAL, NORMAL, MINOR, TRIVIAL).

**`with allure.step("Description"):`**: Breaks test into logical steps shown in the report.

**`allure.attach(data, name, type)`**: Attaches data to the report (logs, screenshots, etc.).

### The Allure Report UI

The Allure report shows:

**Overview**:
- Total tests, pass rate, duration
- Trend graph (if you have historical data)
- Environment info (Python version, OS, etc.)

**Suites**:
- Tests organized by file/class
- Expandable to show individual tests

**Graphs**:
- Tests by feature
- Tests by severity
- Tests by duration

**Timeline**:
- Visual timeline of test execution
- Shows parallel execution

**Behaviors**:
- Tests grouped by feature → story
- Easy to see coverage of each feature

**Test details**:
- Steps with pass/fail status
- Attachments (logs, screenshots)
- Full traceback for failures
- Execution time

## Iteration 5: Integrating Reports in CI

### GitHub Actions with Allure

```yaml
# .github/workflows/test.yml
name: Tests with Allure

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
          cache: 'pip'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest-cov allure-pytest
      
      - name: Run tests
        run: |
          pytest tests/ \
            --alluredir=allure-results \
            --cov=src \
            --cov-report=xml \
            -v
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: allure-results
          path: allure-results/
      
      - name: Generate Allure report
        if: always()
        uses: simple-elf/allure-report-action@master
        with:
          allure_results: allure-results
          allure_history: allure-history
      
      - name: Deploy report to GitHub Pages
        if: always()
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: allure-history
```

This workflow:
1. Runs tests and generates Allure results
2. Uploads results as artifacts
3. Generates the Allure HTML report
4. Deploys it to GitHub Pages

After setup, your report is available at: `https://yourusername.github.io/yourrepo/`

### GitLab CI with Allure

```yaml
# .gitlab-ci.yml
stages:
  - test
  - report

test:
  stage: test
  image: python:3.11
  script:
    - pip install -r requirements.txt
    - pip install pytest-cov allure-pytest
    - pytest tests/ --alluredir=allure-results --cov=src -v
  artifacts:
    paths:
      - allure-results/
    expire_in: 1 week

allure_report:
  stage: report
  image: frankescobar/allure-docker-service:latest
  script:
    - allure generate allure-results -o allure-report --clean
  artifacts:
    paths:
      - allure-report/
    expire_in: 1 month
  only:
    - main
```

## Report Best Practices

### 1. Generate Multiple Report Formats

Different audiences need different formats:
- **JUnit XML**: For CI system integration
- **HTML**: For human browsing
- **Allure**: For detailed analysis and trends

Generate all three:

```bash
$ pytest tests/ \
    --junitxml=junit.xml \
    --html=report.html \
    --self-contained-html \
    --alluredir=allure-results \
    --cov=src \
    --cov-report=html \
    -v
```

### 2. Always Upload Reports as Artifacts

Even if tests pass, save reports for future reference:

```yaml
- name: Upload reports
  if: always()  # Run even if tests fail
  uses: actions/upload-artifact@v3
  with:
    name: test-reports
    path: |
      junit.xml
      report.html
      allure-results/
      htmlcov/
```

### 3. Set Retention Policies

Don't keep reports forever:
- **Test results**: 1 week
- **Coverage reports**: 1 month
- **Allure history**: Keep indefinitely (for trends)

### 4. Make Reports Accessible

Options for sharing reports:
1. **Artifacts**: Download from CI UI
2. **GitHub Pages**: Deploy to a public URL
3. **S3/Cloud Storage**: Upload to a bucket
4. **Report Portal**: Dedicated test reporting service

### 5. Add Context to Reports

Use Allure decorators to add:
- Feature/story grouping
- Severity levels
- Test steps
- Attachments (logs, screenshots)

This makes reports much more useful for debugging and analysis.

### 6. Track Trends Over Time

Keep historical data to see:
- Is test stability improving?
- Are tests getting slower?
- Which tests are flaky?

Allure's history feature does this automatically if you preserve the `allure-history/` directory between runs.

## Automated Testing on Multiple Python Versions (tox)

## The Problem: Version Fragmentation

Your code works on Python 3.11. But your users run:
- Python 3.9 (still common in enterprise)
- Python 3.10 (stable release)
- Python 3.11 (your version)
- Python 3.12 (latest)

Each version has subtle differences. A feature you use might not exist in 3.9. A deprecated API might break in 3.12. **You need to test all versions locally before pushing to CI.**

### What Is Tox?

Tox is a tool for testing Python packages across multiple environments. It:
1. Creates isolated virtual environments for each Python version
2. Installs your package and dependencies in each environment
3. Runs your tests in each environment
4. Reports results for all environments

Think of tox as "CI on your local machine."

### Installing Tox

```bash
$ pip install tox
```

## Iteration 1: The Minimal tox.ini

Tox is configured via `tox.ini` in your project root:

```ini
# tox.ini
[tox]
envlist = py39,py310,py311,py312

[testenv]
deps = 
    pytest
    requests
commands = 
    pytest tests/ -v
```

### Understanding tox.ini Structure

**`[tox]` section**: Global configuration.

**`envlist`**: List of environments to test. `py39` means Python 3.9, `py310` means Python 3.10, etc.

**`[testenv]` section**: Configuration for all test environments.

**`deps`**: Dependencies to install in each environment.

**`commands`**: Commands to run in each environment.

### Running Tox

Simply run `tox`:

```bash
$ tox
py39: install_deps> pip install pytest requests
py39: commands[0]> pytest tests/ -v
======================== test session starts =========================
platform linux -- Python 3.9.16, pytest-7.4.3
collected 15 items

tests/test_payment_processor_extended.py::TestPaymentValidation::test_minimum_amount_accepted PASSED [  6%]
[... all tests ...]
========================= 15 passed in 0.23s =========================
py39: OK ✔ in 3.45 seconds

py310: install_deps> pip install pytest requests
py310: commands[0]> pytest tests/ -v
======================== test session starts =========================
platform linux -- Python 3.10.11, pytest-7.4.3
collected 15 items
[... all tests ...]
========================= 15 passed in 0.21s =========================
py310: OK ✔ in 3.12 seconds

py311: install_deps> pip install pytest requests
py311: commands[0]> pytest tests/ -v
======================== test session starts =========================
platform linux -- Python 3.11.2, pytest-7.4.3
collected 15 items
[... all tests ...]
========================= 15 passed in 0.22s =========================
py311: OK ✔ in 3.08 seconds

py312: install_deps> pip install pytest requests
py312: commands[0]> pytest tests/ -v
======================== test session starts =========================
platform linux -- Python 3.12.0, pytest-7.4.3
collected 15 items
[... all tests ...]
========================= 15 passed in 0.24s =========================
py312: OK ✔ in 3.21 seconds

___________________________ summary ___________________________
  py39: OK ✔ in 3.45 seconds
  py310: OK ✔ in 3.12 seconds
  py311: OK ✔ in 3.08 seconds
  py312: OK ✔ in 3.21 seconds
  congratulations :) (12.86 seconds)
```

Tox automatically:
1. Detected that you have Python 3.9, 3.10, 3.11, and 3.12 installed
2. Created a virtual environment for each
3. Installed dependencies in each
4. Ran tests in each
5. Reported results

### When a Version Fails

Suppose Python 3.9 doesn't support a feature you used:

```python
# src/payment_processor.py (using Python 3.10+ feature)
def process_payment(self, amount: Decimal, card_token: str) -> dict:
    """Process a payment with validation."""
    # Using match/case (Python 3.10+)
    match amount:
        case x if x < self.min_amount:
            return {"success": False, "transaction_id": None, "message": "Too small"}
        case _:
            # Process payment...
            pass
```

Tox shows the failure:

```bash
$ tox
py39: install_deps> pip install pytest requests
py39: commands[0]> pytest tests/ -v
======================== test session starts =========================
platform linux -- Python 3.9.16, pytest-7.4.3
collected 0 items / 1 error

============================== ERRORS ===============================
______________ ERROR collecting src/payment_processor.py ______________
src/payment_processor.py:25: in <module>
    class PaymentProcessor:
src/payment_processor.py:32: in PaymentProcessor
    match amount:
        ^
SyntaxError: invalid syntax
===================== 1 error in 0.05s ==========================
py39: FAIL ✘ in 2.34 seconds

py310: install_deps> pip install pytest requests
py310: commands[0]> pytest tests/ -v
======================== test session starts =========================
platform linux -- Python 3.10.11, pytest-7.4.3
collected 15 items
[... all tests pass ...]
py310: OK ✔ in 3.12 seconds

[... py311 and py312 also pass ...]

___________________________ summary ___________________________
  py39: FAIL ✘ in 2.34 seconds
  py310: OK ✔ in 3.12 seconds
  py311: OK ✔ in 3.08 seconds
  py312: OK ✔ in 3.21 seconds
  evaluation failed :( (11.75 seconds)
```

**Diagnostic Analysis**:

1. **The error**: `SyntaxError: invalid syntax`
   - What this tells us: Python 3.9 can't parse the code

2. **The location**: `match amount:`
   - What this tells us: The `match/case` statement is the problem

3. **Why this happened**:
   - `match/case` was introduced in Python 3.10
   - Python 3.9 doesn't recognize this syntax

**Root cause identified**: Using a feature not available in all target versions.

**Solution**: Either drop Python 3.9 support or use an alternative syntax.

## Iteration 2: Using requirements.txt

Instead of listing dependencies in `tox.ini`, use your existing `requirements.txt`:

```ini
# tox.ini
[tox]
envlist = py39,py310,py311,py312

[testenv]
deps = 
    -rrequirements.txt
    pytest-cov
commands = 
    pytest tests/ --cov=src -v
```

**`-rrequirements.txt`**: Install dependencies from `requirements.txt`.

**Additional deps**: You can still add extra dependencies (like `pytest-cov`) that aren't in `requirements.txt`.

## Iteration 3: Testing with Different Dependency Versions

Sometimes you need to test against multiple versions of a dependency:

```ini
# tox.ini
[tox]
envlist = 
    py{39,310,311,312}-requests{2.28,2.31}

[testenv]
deps = 
    pytest
    requests2.28: requests==2.28.0
    requests2.31: requests==2.31.0
commands = 
    pytest tests/ -v
```

### Understanding Factor Combinations

**`py{39,310,311,312}-requests{2.28,2.31}`**: Creates 8 environments:
- `py39-requests2.28`: Python 3.9 with requests 2.28.0
- `py39-requests2.31`: Python 3.9 with requests 2.31.0
- `py310-requests2.28`: Python 3.10 with requests 2.28.0
- `py310-requests2.31`: Python 3.10 with requests 2.31.0
- ... and so on

**`requests2.28: requests==2.28.0`**: Conditional dependency. Only install `requests==2.28.0` in environments with the `requests2.28` factor.

Running tox now tests all combinations:

```bash
$ tox
py39-requests2.28: OK ✔ in 3.45 seconds
py39-requests2.31: OK ✔ in 3.12 seconds
py310-requests2.28: OK ✔ in 3.08 seconds
py310-requests2.31: OK ✔ in 3.21 seconds
py311-requests2.28: OK ✔ in 3.15 seconds
py311-requests2.31: OK ✔ in 3.09 seconds
py312-requests2.28: OK ✔ in 3.18 seconds
py312-requests2.31: OK ✔ in 3.14 seconds

___________________________ summary ___________________________
  py39-requests2.28: OK ✔ in 3.45 seconds
  py39-requests2.31: OK ✔ in 3.12 seconds
  py310-requests2.28: OK ✔ in 3.08 seconds
  py310-requests2.31: OK ✔ in 3.21 seconds
  py311-requests2.28: OK ✔ in 3.15 seconds
  py311-requests2.31: OK ✔ in 3.09 seconds
  py312-requests2.28: OK ✔ in 3.18 seconds
  py312-requests2.31: OK ✔ in 3.14 seconds
  congratulations :) (25.42 seconds)
```

## Iteration 4: Adding Code Quality Checks

Tox can run more than just tests. Add linting and type checking:

```ini
# tox.ini
[tox]
envlist = 
    py{39,310,311,312}
    lint
    type

[testenv]
deps = 
    -rrequirements.txt
    pytest-cov
commands = 
    pytest tests/ --cov=src --cov-report=term -v

[testenv:lint]
deps = 
    black
    flake8
commands = 
    black --check src/ tests/
    flake8 src/ tests/ --max-line-length=100

[testenv:type]
deps = 
    mypy
    -rrequirements.txt
commands = 
    mypy src/ --ignore-missing-imports
```

### Understanding Named Environments

**`[testenv:lint]`**: A named environment (not tied to a Python version).

**`[testenv:type]`**: Another named environment.

These run in addition to the version-specific environments:

```bash
$ tox
py39: OK ✔ in 3.45 seconds
py310: OK ✔ in 3.12 seconds
py311: OK ✔ in 3.08 seconds
py312: OK ✔ in 3.21 seconds
lint: OK ✔ in 2.15 seconds
type: OK ✔ in 4.32 seconds

___________________________ summary ___________________________
  py39: OK ✔ in 3.45 seconds
  py310: OK ✔ in 3.12 seconds
  py311: OK ✔ in 3.08 seconds
  py312: OK ✔ in 3.21 seconds
  lint: OK ✔ in 2.15 seconds
  type: OK ✔ in 4.32 seconds
  congratulations :) (19.33 seconds)
```

### Running Specific Environments

You don't always want to run all environments. Tox lets you select:

```bash
# Run only Python 3.11 tests
$ tox -e py311

# Run only linting
$ tox -e lint

# Run multiple specific environments
$ tox -e py311,lint,type

# Run all Python 3.11 environments (if you have factors)
$ tox -e py311-requests2.28,py311-requests2.31
```

## Iteration 5: Parallel Execution

Tox can run environments in parallel to save time:

```bash
$ tox -p auto
✔ OK py39 in 3.45 seconds
✔ OK py310 in 3.12 seconds
✔ OK py311 in 3.08 seconds
✔ OK py312 in 3.21 seconds
✔ OK lint in 2.15 seconds
✔ OK type in 4.32 seconds

___________________________ summary ___________________________
  py39: OK ✔ in 3.45 seconds
  py310: OK ✔ in 3.12 seconds
  py311: OK ✔ in 3.08 seconds
  py312: OK ✔ in 3.21 seconds
  lint: OK ✔ in 2.15 seconds
  type: OK ✔ in 4.32 seconds
  congratulations :) (4.32 seconds)  ← Much faster!
```

**`-p auto`**: Run environments in parallel. Tox automatically determines how many to run based on CPU cores.

**Time savings**: Instead of 19.33 seconds (sequential), it takes 4.32 seconds (parallel) — the time of the slowest environment.

## Iteration 6: The Complete Production tox.ini

Here's a full-featured configuration for a production project:

```ini
# tox.ini
[tox]
envlist = 
    py{39,310,311,312}
    lint
    type
    docs
    coverage
skip_missing_interpreters = true

[testenv]
description = Run tests with pytest
deps = 
    -rrequirements.txt
    pytest>=7.4.0
    pytest-cov>=4.1.0
    pytest-xdist>=3.3.0
commands = 
    pytest tests/ \
        --cov=src \
        --cov-report=term-missing \
        --cov-report=xml \
        --cov-report=html \
        -n auto \
        -v
setenv =
    COVERAGE_FILE = .coverage.{envname}

[testenv:lint]
description = Run code quality checks
deps = 
    black>=23.0.0
    flake8>=6.0.0
    isort>=5.12.0
commands = 
    black --check --diff src/ tests/
    isort --check-only --diff src/ tests/
    flake8 src/ tests/ --max-line-length=100 --count --statistics

[testenv:type]
description = Run type checking with mypy
deps = 
    mypy>=1.4.0
    -rrequirements.txt
commands = 
    mypy src/ --ignore-missing-imports --strict

[testenv:docs]
description = Build documentation
deps = 
    sphinx>=7.0.0
    sphinx-rtd-theme>=1.3.0
commands = 
    sphinx-build -W -b html docs/ docs/_build/html

[testenv:coverage]
description = Combine coverage from all test runs
deps = 
    coverage[toml]>=7.2.0
depends = 
    py{39,310,311,312}
commands = 
    coverage combine
    coverage report --fail-under=80
    coverage html

[testenv:clean]
description = Clean up generated files
deps = 
    coverage[toml]>=7.2.0
commands = 
    coverage erase
    python -c "import shutil; shutil.rmtree('htmlcov', ignore_errors=True)"
    python -c "import shutil; shutil.rmtree('.tox', ignore_errors=True)"
    python -c "import shutil; shutil.rmtree('dist', ignore_errors=True)"
    python -c "import shutil; shutil.rmtree('build', ignore_errors=True)"
skip_install = true
```

### Understanding Advanced tox.ini Features

**`skip_missing_interpreters = true`**: Don't fail if a Python version isn't installed. Just skip it.

**`description`**: Human-readable description shown when running tox.

**`setenv`**: Set environment variables for the test environment.

**`COVERAGE_FILE = .coverage.{envname}`**: Create separate coverage files for each environment.

**`depends`**: The `coverage` environment depends on all test environments completing first.

**`skip_install = true`**: Don't install the package (useful for utility environments like `clean`).

### Running the Complete Suite

```bash
$ tox -p auto
✔ OK py39 in 3.45 seconds
✔ OK py310 in 3.12 seconds
✔ OK py311 in 3.08 seconds
✔ OK py312 in 3.21 seconds
✔ OK lint in 2.15 seconds
✔ OK type in 4.32 seconds
✔ OK docs in 5.67 seconds
✔ OK coverage in 1.23 seconds

___________________________ summary ___________________________
  py39: OK ✔ in 3.45 seconds
  py310: OK ✔ in 3.12 seconds
  py311: OK ✔ in 3.08 seconds
  py312: OK ✔ in 3.21 seconds
  lint: OK ✔ in 2.15 seconds
  type: OK ✔ in 4.32 seconds
  docs: OK ✔ in 5.67 seconds
  coverage: OK ✔ in 1.23 seconds
  congratulations :) (5.67 seconds)
```

## Integrating Tox with CI

### GitHub Actions with Tox

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        python-version: ['3.9', '3.10', '3.11', '3.12']
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v4
        with:
          python-version: ${{ matrix.python-version }}
      
      - name: Install tox
        run: pip install tox tox-gh-actions
      
      - name: Run tox
        run: tox
```

**`tox-gh-actions`**: A plugin that automatically maps GitHub Actions matrix to tox environments.

With this plugin, you don't need to specify `-e py39` manually. Tox detects the Python version from the matrix and runs the appropriate environment.

### GitLab CI with Tox

```yaml
# .gitlab-ci.yml
stages:
  - test

.test_template: &test_template
  stage: test
  before_script:
    - pip install tox
  script:
    - tox -e ${TOX_ENV}

test_py39:
  <<: *test_template
  image: python:3.9
  variables:
    TOX_ENV: py39

test_py310:
  <<: *test_template
  image: python:3.10
  variables:
    TOX_ENV: py310

test_py311:
  <<: *test_template
  image: python:3.11
  variables:
    TOX_ENV: py311

test_py312:
  <<: *test_template
  image: python:3.12
  variables:
    TOX_ENV: py312

lint:
  stage: test
  image: python:3.11
  before_script:
    - pip install tox
  script:
    - tox -e lint

type:
  stage: test
  image: python:3.11
  before_script:
    - pip install tox
  script:
    - tox -e type
```

## Tox Best Practices

### 1. Use tox Locally Before Pushing

Run `tox` before committing to catch issues early:

```bash
# Quick check before commit
$ tox -e py311,lint

# Full check before push
$ tox -p auto
```

### 2. Pin Dependency Versions

Use exact versions in `requirements.txt` to ensure reproducibility:

```text
# requirements.txt
requests==2.31.0
pytest==7.4.3
pytest-cov==4.1.0
```

### 3. Use Factors for Combinations

Test multiple dependency versions with factors:

```ini
[tox]
envlist = py{39,310,311,312}-django{3.2,4.0,4.1}
```

### 4. Separate Test and Quality Environments

Don't mix tests and linting in the same environment:

```ini
[testenv]
commands = pytest tests/

[testenv:lint]
commands = black --check src/ tests/
```

### 5. Use Parallel Execution

Always use `-p auto` for faster feedback:

```bash
$ tox -p auto
```

### 6. Skip Missing Interpreters

Don't fail if a Python version isn't installed:

```ini
[tox]
skip_missing_interpreters = true
```

### 7. Clean Up Between Runs

Periodically clean tox environments:

```bash
# Remove all tox environments
$ tox -e clean

# Or manually
$ rm -rf .tox/
```

## The Journey: From Local Testing to Full Automation

| Stage                  | Tool                | What It Tests                                  |
| ---------------------- | ------------------- | ---------------------------------------------- |
| **Local development**  | pytest              | Current Python version, current dependencies   |
| **Pre-commit**         | tox -e py311,lint   | Quick check before committing                  |
| **Pre-push**           | tox -p auto         | All Python versions, all checks                |
| **CI (GitHub Actions)**| Matrix + tox        | All versions, all OSes, all dependency combos  |
| **Nightly builds**     | tox + cron          | Latest dependencies, catch breaking changes    |

### The Complete Workflow

1. **Write code**: Test with `pytest tests/ -v`
2. **Before commit**: Run `tox -e py311,lint`
3. **Before push**: Run `tox -p auto`
4. **Push to CI**: GitHub Actions runs full matrix
5. **Review reports**: Check test results, coverage, and quality metrics
6. **Merge**: Only if all checks pass

This multi-layered approach catches issues at the earliest possible stage, saving time and preventing bugs from reaching production.
