# Chapter 3: Setting Up Your Testing Environment

## Project Structure Best Practices

## The Foundation: Why Structure Matters

Before writing a single test, you need to answer a deceptively simple question: **Where do the tests go?**

This isn't just about organization—it's about making pytest work effortlessly. A well-structured project means pytest finds your tests automatically, imports work without path manipulation, and your team can navigate the codebase intuitively.

Let's build a real project structure from scratch and see how each decision affects our testing workflow.

## The Reference Project: A Payment Processing System

We'll use a concrete example throughout this chapter: a payment processing library called `payflow`. This is substantial enough to demonstrate real-world structure challenges but simple enough to understand immediately.

Our `payflow` library will have:
- Core payment processing logic
- Database models
- API client for external payment gateways
- Utility functions for validation

Here's what most beginners try first:

```text
payflow_project/
├── payflow.py          # Everything in one file
└── test_payflow.py     # All tests in one file
```

This works for exactly one day. Then you add a second module, and suddenly:

```python
# test_payflow.py
from payflow import process_payment  # Works fine

# But now you add database.py...
from database import save_transaction  # ModuleNotFoundError!
```

### The Problem: Python's Import System

Python needs to know where to find your modules. When you run `pytest`, it doesn't automatically know that `database.py` is part of your project. You need a **package structure**.

## Iteration 1: Creating a Package Structure

Let's transform our flat structure into a proper Python package:

```text
payflow_project/
├── payflow/                 # Package directory
│   ├── __init__.py         # Makes it a package
│   ├── core.py             # Payment processing
│   ├── database.py         # Database operations
│   ├── api_client.py       # External API calls
│   └── validators.py       # Input validation
└── tests/
    ├── __init__.py
    └── test_core.py
```

The `__init__.py` files are crucial—they tell Python "this directory is a package." Even if they're empty, they must exist.

Let's create this structure and write our first test:

```bash
mkdir -p payflow_project/payflow
mkdir -p payflow_project/tests
cd payflow_project
touch payflow/__init__.py tests/__init__.py
```

```python
# payflow/core.py
def process_payment(amount, currency="USD"):
    """Process a payment transaction."""
    if amount <= 0:
        raise ValueError("Amount must be positive")
    if currency not in ["USD", "EUR", "GBP"]:
        raise ValueError(f"Unsupported currency: {currency}")
    
    # Simulate payment processing
    transaction_id = f"TXN-{amount}-{currency}"
    return {
        "transaction_id": transaction_id,
        "amount": amount,
        "currency": currency,
        "status": "completed"
    }
```

```python
# tests/test_core.py
from payflow.core import process_payment

def test_successful_payment():
    result = process_payment(100.00, "USD")
    assert result["status"] == "completed"
    assert result["amount"] == 100.00
```

Now run pytest from the project root:

```bash
cd payflow_project
pytest
```

**Expected output**:

```text
============================= test session starts ==============================
collected 1 item

tests/test_core.py .                                                     [100%]

============================== 1 passed in 0.01s ===============================
```

### What Just Happened?

Pytest automatically:
1. Found the `tests/` directory
2. Discovered `test_core.py` (starts with `test_`)
3. Found `test_successful_payment()` (starts with `test_`)
4. Imported `payflow.core` successfully because we're running from the project root

## Iteration 2: Mirroring Source Structure in Tests

As our project grows, we add more modules. Our tests should mirror this structure:

```text
payflow_project/
├── payflow/
│   ├── __init__.py
│   ├── core.py
│   ├── database.py
│   ├── api_client.py
│   └── validators.py
└── tests/
    ├── __init__.py
    ├── test_core.py          # Tests for core.py
    ├── test_database.py      # Tests for database.py
    ├── test_api_client.py    # Tests for api_client.py
    └── test_validators.py    # Tests for validators.py
```

**Why mirror the structure?**

1. **Discoverability**: Finding tests for `payflow/database.py` is instant—look in `tests/test_database.py`
2. **Maintenance**: When you refactor `core.py`, you know exactly which test file to update
3. **Team coordination**: No confusion about where new tests should go

Let's add a database module and its tests:

```python
# payflow/database.py
class TransactionDB:
    """Simple in-memory transaction storage."""
    
    def __init__(self):
        self.transactions = {}
    
    def save_transaction(self, transaction_id, data):
        """Save a transaction to the database."""
        if not transaction_id:
            raise ValueError("Transaction ID cannot be empty")
        
        self.transactions[transaction_id] = data
        return transaction_id
    
    def get_transaction(self, transaction_id):
        """Retrieve a transaction by ID."""
        if transaction_id not in self.transactions:
            raise KeyError(f"Transaction {transaction_id} not found")
        return self.transactions[transaction_id]
```

```python
# tests/test_database.py
from payflow.database import TransactionDB

def test_save_and_retrieve_transaction():
    db = TransactionDB()
    
    transaction_data = {
        "amount": 100.00,
        "currency": "USD",
        "status": "completed"
    }
    
    txn_id = db.save_transaction("TXN-001", transaction_data)
    retrieved = db.get_transaction(txn_id)
    
    assert retrieved["amount"] == 100.00
    assert retrieved["status"] == "completed"
```

Run pytest again:

```bash
pytest
```

```text
============================= test session starts ==============================
collected 2 items

tests/test_core.py .                                                     [ 50%]
tests/test_database.py .                                                 [100%]

============================== 2 passed in 0.02s ===============================
```

Pytest found both test files automatically. No configuration needed.

## Iteration 3: Organizing Tests by Type

Real projects have different kinds of tests:
- **Unit tests**: Test individual functions in isolation
- **Integration tests**: Test how components work together
- **End-to-end tests**: Test complete workflows

Let's organize by test type:

```text
payflow_project/
├── payflow/
│   └── ...
└── tests/
    ├── __init__.py
    ├── unit/
    │   ├── __init__.py
    │   ├── test_core.py
    │   ├── test_database.py
    │   └── test_validators.py
    ├── integration/
    │   ├── __init__.py
    │   ├── test_payment_flow.py
    │   └── test_database_integration.py
    └── e2e/
        ├── __init__.py
        └── test_complete_payment.py
```

Now we can run specific test categories:

```bash
pytest tests/unit/              # Only unit tests
pytest tests/integration/       # Only integration tests
pytest tests/e2e/              # Only end-to-end tests
```

Let's create an integration test that combines our core and database modules:

```python
# tests/integration/test_payment_flow.py
from payflow.core import process_payment
from payflow.database import TransactionDB

def test_payment_is_saved_to_database():
    """Integration test: payment processing + database storage."""
    db = TransactionDB()
    
    # Process payment
    result = process_payment(150.00, "EUR")
    
    # Save to database
    txn_id = result["transaction_id"]
    db.save_transaction(txn_id, result)
    
    # Verify it's retrievable
    saved = db.get_transaction(txn_id)
    assert saved["amount"] == 150.00
    assert saved["currency"] == "EUR"
    assert saved["status"] == "completed"
```

Run just the integration tests:

```bash
pytest tests/integration/ -v
```

```text
============================= test session starts ==============================
collected 1 item

tests/integration/test_payment_flow.py::test_payment_is_saved_to_database PASSED [100%]

============================== 1 passed in 0.01s ===============================
```

## Iteration 4: Adding Shared Test Utilities

As tests grow, you'll need shared utilities—test data builders, custom assertions, helper functions. Where do these go?

**Anti-pattern**: Putting them in test files leads to duplication.

**Solution**: Create a `tests/helpers/` or `tests/conftest.py` (we'll cover conftest.py in detail in Chapter 4).

For now, let's add shared test data:

```text
payflow_project/
├── payflow/
│   └── ...
└── tests/
    ├── __init__.py
    ├── helpers/
    │   ├── __init__.py
    │   └── builders.py      # Test data builders
    ├── unit/
    │   └── ...
    └── integration/
        └── ...
```

```python
# tests/helpers/builders.py
"""Test data builders for consistent test data creation."""

def build_transaction(amount=100.00, currency="USD", status="completed"):
    """Build a transaction dictionary with sensible defaults."""
    return {
        "transaction_id": f"TXN-{amount}-{currency}",
        "amount": amount,
        "currency": currency,
        "status": status
    }

def build_failed_transaction(amount=100.00, currency="USD"):
    """Build a failed transaction for error testing."""
    return build_transaction(amount, currency, status="failed")
```

Now tests can use these builders:

```python
# tests/unit/test_database.py
from payflow.database import TransactionDB
from tests.helpers.builders import build_transaction

def test_save_transaction_with_builder():
    db = TransactionDB()
    transaction = build_transaction(amount=200.00, currency="GBP")
    
    txn_id = db.save_transaction(transaction["transaction_id"], transaction)
    retrieved = db.get_transaction(txn_id)
    
    assert retrieved["amount"] == 200.00
    assert retrieved["currency"] == "GBP"
```

## The Complete Structure: Production-Ready

Here's the final, production-ready structure:

```text
payflow_project/
├── README.md
├── setup.py                 # Package installation (we'll add this in 3.3)
├── requirements.txt         # Production dependencies
├── requirements-dev.txt     # Development dependencies (pytest, etc.)
├── .gitignore
├── payflow/                 # Source package
│   ├── __init__.py
│   ├── core.py
│   ├── database.py
│   ├── api_client.py
│   └── validators.py
└── tests/                   # Test package
    ├── __init__.py
    ├── conftest.py          # Shared fixtures (Chapter 4)
    ├── helpers/
    │   ├── __init__.py
    │   └── builders.py
    ├── unit/
    │   ├── __init__.py
    │   ├── test_core.py
    │   ├── test_database.py
    │   ├── test_api_client.py
    │   └── test_validators.py
    ├── integration/
    │   ├── __init__.py
    │   ├── test_payment_flow.py
    │   └── test_database_integration.py
    └── e2e/
        ├── __init__.py
        └── test_complete_payment.py
```

## Decision Framework: Choosing Your Structure

| Project Size | Recommended Structure | Rationale |
|--------------|----------------------|-----------|
| Single module (&lt;500 lines) | Flat: `project.py` + `test_project.py` | Simplicity wins |
| Small library (2-5 modules) | Package + mirrored tests | Discoverability matters |
| Medium project (6-20 modules) | Package + tests by type | Need to run test categories separately |
| Large project (20+ modules) | Package + tests by type + helpers | Shared utilities become essential |

## Key Principles

1. **Mirror source structure in tests**: `payflow/core.py` → `tests/test_core.py` or `tests/unit/test_core.py`
2. **Use `__init__.py` everywhere**: Makes directories importable
3. **Run pytest from project root**: Ensures imports work consistently
4. **Organize by test type when needed**: Unit, integration, e2e
5. **Create helpers for shared utilities**: Avoid duplication across test files

## Common Failure Mode: Import Errors

**Symptom**: `ModuleNotFoundError: No module named 'payflow'`

**Diagnostic clues**:
- You're running pytest from the wrong directory
- Missing `__init__.py` files
- Package not installed in development mode (see Section 3.3)

**Solution**: Always run pytest from the project root where your package directory is visible.

## Creating a tests/ Directory

## The Question: Where Exactly Do Tests Live?

You've decided to create a `tests/` directory. But should it be:
- Inside your package? (`payflow/tests/`)
- Next to your package? (`tests/` at project root)
- Somewhere else entirely?

This isn't just aesthetics—it affects imports, packaging, and deployment.

## The Reference Implementation: Tests Inside the Package

Let's start with what seems intuitive—putting tests inside the package:

```text
payflow_project/
└── payflow/
    ├── __init__.py
    ├── core.py
    ├── database.py
    └── tests/              # Tests inside package
        ├── __init__.py
        ├── test_core.py
        └── test_database.py
```

```python
# payflow/tests/test_core.py
from payflow.core import process_payment

def test_process_payment():
    result = process_payment(100.00)
    assert result["status"] == "completed"
```

Run pytest:

```bash
pytest
```

```text
============================= test session starts ==============================
collected 1 item

payflow/tests/test_core.py .                                             [100%]

============================== 1 passed in 0.01s ===============================
```

This works! Tests run successfully. So what's the problem?

### The Hidden Problem: Tests in Production

Let's package this for distribution:

```python
# setup.py
from setuptools import setup, find_packages

setup(
    name="payflow",
    version="1.0.0",
    packages=find_packages(),  # Finds all packages
)
```

```bash
python setup.py sdist
tar -tzf dist/payflow-1.0.0.tar.gz
```

```text
payflow-1.0.0/
payflow-1.0.0/payflow/
payflow-1.0.0/payflow/__init__.py
payflow-1.0.0/payflow/core.py
payflow-1.0.0/payflow/database.py
payflow-1.0.0/payflow/tests/          # ⚠️ Tests included!
payflow-1.0.0/payflow/tests/__init__.py
payflow-1.0.0/payflow/tests/test_core.py
payflow-1.0.0/payflow/tests/test_database.py
```

### Diagnostic Analysis: Why This Is Problematic

**The issue**: Your tests are now part of the installed package. When users install `payflow`, they get:
- Your production code (good)
- Your entire test suite (bad)
- Test dependencies like pytest (bad)
- Test data files (bad)

**Consequences**:
1. **Bloated installations**: Users download unnecessary test code
2. **Namespace pollution**: `from payflow.tests import ...` is possible but meaningless
3. **Dependency confusion**: Test dependencies might conflict with user code
4. **Security concerns**: Test code might contain credentials or test data

**What we need**: Tests that run during development but don't ship to users.

## Iteration 1: Tests Outside the Package

Move tests to the project root, next to the package:

```text
payflow_project/
├── payflow/                # Source package
│   ├── __init__.py
│   ├── core.py
│   └── database.py
└── tests/                  # Tests outside package
    ├── __init__.py
    ├── test_core.py
    └── test_database.py
```

The imports remain identical:

```python
# tests/test_core.py
from payflow.core import process_payment  # Same import!

def test_process_payment():
    result = process_payment(100.00)
    assert result["status"] == "completed"
```

Now package it:

```bash
python setup.py sdist
tar -tzf dist/payflow-1.0.0.tar.gz
```

```text
payflow-1.0.0/
payflow-1.0.0/payflow/
payflow-1.0.0/payflow/__init__.py
payflow-1.0.0/payflow/core.py
payflow-1.0.0/payflow/database.py
# ✅ No tests/ directory!
```

**Perfect**. Tests are excluded from the distribution automatically because `find_packages()` only finds packages inside the source directory.

### When to Use Each Approach

| Approach | Use When | Avoid When |
|----------|----------|------------|
| Tests inside package (`payflow/tests/`) | Never recommended | Always—tests shouldn't ship to users |
| Tests outside package (`tests/` at root) | Always—this is the standard | You're building a single-file script |

## Iteration 2: The `__init__.py` Question

Should `tests/__init__.py` exist?

**Two schools of thought**:

1. **With `__init__.py`**: Makes `tests/` a package
2. **Without `__init__.py`**: Keeps `tests/` as a simple directory

Let's test both:

```text
# Approach 1: tests/ is a package
tests/
├── __init__.py          # Present
├── test_core.py
└── helpers/
    ├── __init__.py
    └── builders.py
```

```python
# tests/test_core.py
from tests.helpers.builders import build_transaction  # Works

def test_with_builder():
    txn = build_transaction()
    assert txn["amount"] == 100.00
```

```text
# Approach 2: tests/ is just a directory
tests/
├── test_core.py         # No __init__.py
└── helpers/
    ├── __init__.py
    └── builders.py
```

```python
# tests/test_core.py
from helpers.builders import build_transaction  # Also works

def test_with_builder():
    txn = build_transaction()
    assert txn["amount"] == 100.00
```

Both work! Pytest adds the `tests/` directory to `sys.path` automatically.

### The Recommendation: Skip `tests/__init__.py`

**Why?**
1. **Simpler imports**: `from helpers.builders import ...` instead of `from tests.helpers.builders import ...`
2. **Clearer intent**: Tests aren't meant to be imported as a package
3. **Less confusion**: New developers don't wonder why tests are a package

**Exception**: Keep `__init__.py` in subdirectories like `tests/helpers/` if you want to import from them.

## Iteration 3: Handling Test Data Files

Real tests need data files—JSON fixtures, CSV files, images. Where do these go?

```text
tests/
├── test_core.py
├── test_database.py
├── data/                    # Test data directory
│   ├── valid_transaction.json
│   ├── invalid_transaction.json
│   └── sample_users.csv
└── helpers/
    └── builders.py
```

Loading test data:

```python
# tests/test_core.py
import json
from pathlib import Path

def test_process_payment_from_file():
    # Get the directory containing this test file
    test_dir = Path(__file__).parent
    data_file = test_dir / "data" / "valid_transaction.json"
    
    with open(data_file) as f:
        transaction_data = json.load(f)
    
    # Use the data in your test
    assert transaction_data["amount"] == 100.00
```

**Key technique**: Use `Path(__file__).parent` to locate files relative to the test file. This works regardless of where pytest is run from.

### Test Data Organization

```text
tests/
├── data/
│   ├── transactions/        # Organized by domain
│   │   ├── valid.json
│   │   └── invalid.json
│   ├── users/
│   │   └── sample_users.csv
│   └── fixtures/            # Reusable test fixtures
│       └── database_seed.sql
├── unit/
│   └── test_core.py
└── integration/
    └── test_payment_flow.py
```

## The Complete tests/ Directory Structure

Here's the production-ready structure:

```text
payflow_project/
├── payflow/                 # Source package
│   ├── __init__.py
│   ├── core.py
│   └── database.py
└── tests/                   # Tests directory (NOT a package)
    ├── conftest.py          # Shared fixtures (Chapter 4)
    ├── data/                # Test data files
    │   ├── transactions/
    │   │   ├── valid.json
    │   │   └── invalid.json
    │   └── users/
    │       └── sample_users.csv
    ├── helpers/             # Test utilities (IS a package)
    │   ├── __init__.py
    │   └── builders.py
    ├── unit/
    │   ├── test_core.py
    │   └── test_database.py
    └── integration/
        └── test_payment_flow.py
```

## Decision Framework: tests/ Directory Placement

| Question | Answer | Rationale |
|----------|--------|-----------|
| Inside or outside package? | Outside (at project root) | Tests shouldn't ship to users |
| Should tests/ have `__init__.py`? | No | Simpler imports, clearer intent |
| Should subdirectories have `__init__.py`? | Yes, if you import from them | Makes them importable packages |
| Where do test data files go? | `tests/data/` | Keeps test files separate from code |
| How to reference test data? | `Path(__file__).parent / "data"` | Works from any directory |

## Common Failure Modes

### Symptom: Tests run but aren't included in coverage

**Pytest output**:

```text
============================= test session starts ==============================
collected 0 items

============================ no tests ran in 0.01s =============================
```

**Diagnostic clues**:
- Pytest found no test files
- You might be running pytest from the wrong directory
- Test files might not follow naming conventions

**Solution**: Run pytest from project root, ensure test files start with `test_`.

### Symptom: Import errors when loading test data

**Error**:

```text
FileNotFoundError: [Errno 2] No such file or directory: 'data/valid.json'
```

**Root cause**: Using relative paths that depend on current working directory.

**Solution**: Use `Path(__file__).parent` for test-file-relative paths.

## Key Principles

1. **Tests outside the package**: Never ship tests to users
2. **No `tests/__init__.py`**: Tests aren't a package
3. **Subdirectories can be packages**: If you need to import from them
4. **Test data in `tests/data/`**: Separate data from code
5. **Use `Path(__file__).parent`**: For reliable file references

## Using Virtual Environments

## The Problem: Dependency Chaos

You're working on two Python projects:
- **Project A** needs `pytest==7.4.0`
- **Project B** needs `pytest==6.2.5`

Install pytest for Project A:

```bash
pip install pytest==7.4.0
```

Now switch to Project B:

```bash
cd ../project_b
pytest
```

```text
ERROR: This version of pytest requires pytest-7.4.0, but you have pytest-6.2.5
```

**The conflict**: You can't have two versions of the same package installed globally. Every project shares the same Python environment.

### Diagnostic Analysis: Why Global Installation Fails

When you run `pip install`, packages go into Python's global `site-packages` directory:

```bash
python -c "import site; print(site.getsitepackages())"
```

```text
['/usr/local/lib/python3.11/site-packages']
```

**The problem**:
1. All projects share this directory
2. Only one version of each package can exist
3. Installing a new version overwrites the old one
4. Different projects can't have different dependencies

**What we need**: Isolated environments where each project has its own dependencies.

## The Solution: Virtual Environments

A **virtual environment** is an isolated Python installation. Each project gets its own:
- Python interpreter (linked to the system Python)
- `site-packages` directory (independent package storage)
- `pip` installation (project-specific package manager)

## Iteration 1: Creating Your First Virtual Environment

Let's create a virtual environment for our `payflow` project:

```bash
cd payflow_project
python -m venv venv
```

**What just happened?**

The `python -m venv venv` command:
1. Created a `venv/` directory
2. Copied the Python interpreter into it
3. Created an isolated `site-packages` directory
4. Installed `pip` and `setuptools` in the isolated environment

Let's examine the structure:

```bash
ls -la venv/
```

```text
venv/
├── bin/                    # Executables (Linux/Mac)
│   ├── python             # Isolated Python interpreter
│   ├── pip                # Isolated pip
│   └── activate           # Activation script
├── include/               # C headers
├── lib/                   # Isolated packages
│   └── python3.11/
│       └── site-packages/ # Project-specific packages
└── pyvenv.cfg            # Configuration
```

The environment exists but isn't active yet. Check which Python you're using:

```bash
which python
```

```text
/usr/bin/python              # Still using system Python
```

### Activating the Virtual Environment

To use the isolated environment, you must **activate** it:

```bash
source venv/bin/activate     # Linux/Mac
# or
venv\Scripts\activate        # Windows
```

Your prompt changes to show the active environment:

```text
(venv) user@machine:~/payflow_project$
```

Now check which Python you're using:

```bash
which python
```

```text
/home/user/payflow_project/venv/bin/python  # Using virtual environment!
```

### Installing Packages in the Virtual Environment

With the environment activated, install pytest:

```bash
pip install pytest
```

```text
Collecting pytest
  Downloading pytest-7.4.3-py3-none-any.whl (325 kB)
Installing collected packages: pytest
Successfully installed pytest-7.4.3
```

Where did it install?

```bash
pip show pytest
```

```text
Name: pytest
Version: 7.4.3
Location: /home/user/payflow_project/venv/lib/python3.11/site-packages
```

**Perfect**. Pytest is installed in the project's isolated environment, not globally.

### Verifying Isolation

Let's prove the environments are truly isolated. Deactivate the virtual environment:

```bash
deactivate
```

Your prompt returns to normal:

```text
user@machine:~/payflow_project$
```

Try to run pytest:

```bash
pytest
```

```text
bash: pytest: command not found
```

**Excellent**. Pytest only exists inside the virtual environment. Reactivate to use it:

```bash
source venv/bin/activate
pytest --version
```

```text
pytest 7.4.3
```

## Iteration 2: Managing Dependencies with requirements.txt

You've installed pytest, but how do other developers know what to install? How do you remember six months from now?

**Solution**: Document dependencies in `requirements.txt`.

### Creating requirements.txt

List your current packages:

```bash
pip freeze
```

```text
iniconfig==2.0.0
packaging==23.2
pluggy==1.3.0
pytest==7.4.3
```

Save this to a file:

```bash
pip freeze > requirements.txt
```

Now `requirements.txt` contains:

```text
iniconfig==2.0.0
packaging==23.2
pluggy==1.3.0
pytest==7.4.3
```

### Installing from requirements.txt

A new developer clones your project:

```bash
git clone https://github.com/yourname/payflow.git
cd payflow
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

**Result**: They get exactly the same package versions you used.

### Separating Development and Production Dependencies

Your project needs:
- **Production dependencies**: Required to run the code (e.g., `requests`, `sqlalchemy`)
- **Development dependencies**: Required to develop/test (e.g., `pytest`, `black`, `mypy`)

Users who install your package shouldn't need pytest. Create two files:

```text
# requirements.txt (production)
requests==2.31.0
sqlalchemy==2.0.23
```

```text
# requirements-dev.txt (development)
-r requirements.txt          # Include production dependencies
pytest==7.4.3
pytest-cov==4.1.0
black==23.11.0
mypy==1.7.1
```

The `-r requirements.txt` line includes production dependencies in the development file.

Install development dependencies:

```bash
pip install -r requirements-dev.txt
```

This installs both production and development packages.

## Iteration 3: Installing Your Package in Development Mode

Right now, to import `payflow`, pytest must be run from the project root. This is fragile. Let's make `payflow` properly installable.

### Creating setup.py

Create a minimal `setup.py`:

```python
# setup.py
from setuptools import setup, find_packages

setup(
    name="payflow",
    version="1.0.0",
    packages=find_packages(),
    install_requires=[
        "requests>=2.31.0",
        "sqlalchemy>=2.0.23",
    ],
    extras_require={
        "dev": [
            "pytest>=7.4.3",
            "pytest-cov>=4.1.0",
            "black>=23.11.0",
            "mypy>=1.7.1",
        ]
    }
)
```

### Installing in Editable Mode

Install your package in **editable mode** (also called **development mode**):

```bash
pip install -e .
```

The `-e` flag means "editable"—changes to your source code are immediately reflected without reinstalling.

**What this does**:
1. Creates a link from `site-packages` to your source directory
2. Makes `payflow` importable from anywhere
3. Allows you to edit code and see changes immediately

Verify it worked:

```bash
python -c "import payflow; print(payflow.__file__)"
```

```text
/home/user/payflow_project/payflow/__init__.py
```

Now you can run pytest from any directory:

```bash
cd /tmp
pytest /home/user/payflow_project/tests/
```

```text
============================= test session starts ==============================
collected 2 items

/home/user/payflow_project/tests/test_core.py .                         [ 50%]
/home/user/payflow_project/tests/test_database.py .                     [100%]

============================== 2 passed in 0.02s ===============================
```

**Why this matters**: CI/CD systems often run tests from different directories. Editable installation ensures imports always work.

### Installing with Development Dependencies

Install your package with development extras:

```bash
pip install -e ".[dev]"
```

This installs:
1. Your package in editable mode
2. Production dependencies from `install_requires`
3. Development dependencies from `extras_require["dev"]`

## The Complete Workflow: From Clone to Test

Here's the complete workflow a new developer follows:

```bash
# 1. Clone the repository
git clone https://github.com/yourname/payflow.git
cd payflow

# 2. Create virtual environment
python -m venv venv

# 3. Activate virtual environment
source venv/bin/activate  # Linux/Mac
# or
venv\Scripts\activate     # Windows

# 4. Install package with development dependencies
pip install -e ".[dev]"

# 5. Run tests
pytest

# 6. Start developing
# Edit code, run tests, repeat
```

## Project Structure with Virtual Environment

Your complete project now looks like:

```text
payflow_project/
├── venv/                    # Virtual environment (gitignored)
│   ├── bin/
│   ├── lib/
│   └── ...
├── payflow/                 # Source package
│   ├── __init__.py
│   ├── core.py
│   └── database.py
├── tests/
│   ├── test_core.py
│   └── test_database.py
├── setup.py                 # Package configuration
├── requirements.txt         # Production dependencies
├── requirements-dev.txt     # Development dependencies
├── .gitignore              # Ignore venv/
└── README.md
```

### Critical: Add venv/ to .gitignore

Never commit your virtual environment to version control:

```text
# .gitignore
venv/
*.pyc
__pycache__/
.pytest_cache/
*.egg-info/
dist/
build/
```

**Why?**
1. Virtual environments are large (100+ MB)
2. They're platform-specific (Linux venv won't work on Windows)
3. They're easily recreated from `requirements.txt`

## Decision Framework: Virtual Environment Choices

| Question | Recommendation | Alternative |
|----------|---------------|-------------|
| Which tool to use? | `venv` (built-in) | `virtualenv`, `conda`, `poetry` |
| Where to create it? | Project root (`venv/`) | Anywhere, but document it |
| What to name it? | `venv` or `.venv` | Any name, but be consistent |
| Commit to git? | Never | N/A |
| One venv per project? | Yes | Shared venvs cause conflicts |

## Common Failure Modes

### Symptom: "pytest: command not found" after creating venv

**Diagnostic clues**:
- Virtual environment exists but isn't activated
- Prompt doesn't show `(venv)`

**Solution**: Run `source venv/bin/activate` (Linux/Mac) or `venv\Scripts\activate` (Windows)

### Symptom: Tests pass locally but fail in CI

**Pytest output in CI**:

```text
ModuleNotFoundError: No module named 'payflow'
```

**Root cause**: Package not installed in CI environment

**Solution**: Add installation step to CI configuration:

```yaml
# .github/workflows/test.yml
- name: Install dependencies
  run: |
    python -m pip install --upgrade pip
    pip install -e ".[dev]"
```

### Symptom: Import works in one terminal but not another

**Diagnostic clues**:
- Different terminals show different `which python` output
- One terminal has `(venv)` in prompt, other doesn't

**Root cause**: Virtual environment activated in one terminal but not the other

**Solution**: Each terminal session needs its own activation. Consider using tools like `direnv` for automatic activation.

## Key Principles

1. **One virtual environment per project**: Isolate dependencies
2. **Always activate before working**: `source venv/bin/activate`
3. **Document dependencies**: Use `requirements.txt` or `setup.py`
4. **Install in editable mode**: `pip install -e .` for development
5. **Never commit venv/**: Add to `.gitignore`
6. **Separate dev and prod dependencies**: Use `requirements-dev.txt` or `extras_require`

## Pytest Configuration Files (pytest.ini, setup.cfg, pyproject.toml)

## The Problem: Repeating Command-Line Options

You're running pytest with the same options every time:

```bash
pytest -v --tb=short --strict-markers tests/
```

This gets tedious. You want these options to be the default. But how?

**The solution**: Configuration files. Pytest reads settings from configuration files, so you type less and ensure consistency across your team.

## The Three Configuration File Options

Pytest supports three configuration file formats:

1. **`pytest.ini`**: Pytest-specific, INI format
2. **`setup.cfg`**: Shared with setuptools, INI format
3. **`pyproject.toml`**: Modern Python standard, TOML format

Let's explore each with our `payflow` project.

## Iteration 1: Using pytest.ini

Create `pytest.ini` in your project root:

```ini
# pytest.ini
[pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
```

Now run pytest without arguments:

```bash
pytest
```

```text
============================= test session starts ==============================
collected 2 items

tests/test_core.py .                                                     [ 50%]
tests/test_database.py .                                                 [100%]

============================== 2 passed in 0.02s ===============================
```

**What happened?**

Pytest read `pytest.ini` and applied the configuration:
- `testpaths = tests`: Only look for tests in the `tests/` directory
- `python_files = test_*.py`: Only collect files starting with `test_`
- `python_classes = Test*`: Only collect classes starting with `Test`
- `python_functions = test_*`: Only collect functions starting with `test_`

These are actually pytest's defaults, but now they're explicit and documented.

### Adding Useful Options

Let's add options we use frequently:

```ini
# pytest.ini
[pytest]
# Test discovery
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*

# Output options
addopts = 
    -v
    --tb=short
    --strict-markers
    --strict-config

# Markers
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    integration: marks tests as integration tests
    unit: marks tests as unit tests
```

Now run pytest:

```bash
pytest
```

```text
============================= test session starts ==============================
collected 2 items

tests/test_core.py::test_successful_payment PASSED                       [ 50%]
tests/test_database.py::test_save_and_retrieve_transaction PASSED        [100%]

============================== 2 passed in 0.02s ===============================
```

Notice the output is now verbose (`-v`) automatically—we didn't type it!

### Understanding addopts

The `addopts` option adds command-line arguments automatically. Let's break down what we added:

- **`-v`**: Verbose output (show test names)
- **`--tb=short`**: Shorter tracebacks (less noise on failures)
- **`--strict-markers`**: Fail if undefined markers are used
- **`--strict-config`**: Fail if configuration has errors

These options now apply to every pytest run.

## Iteration 2: Using setup.cfg

If you already have `setup.cfg` for setuptools, you can add pytest configuration there:

```ini
# setup.cfg
[metadata]
name = payflow
version = 1.0.0

[options]
packages = find:
install_requires =
    requests>=2.31.0

[options.extras_require]
dev =
    pytest>=7.4.3
    pytest-cov>=4.1.0

[tool:pytest]
testpaths = tests
addopts =
    -v
    --tb=short
    --strict-markers
markers =
    slow: marks tests as slow
    integration: marks tests as integration tests
```

**Key difference**: The section is `[tool:pytest]` instead of `[pytest]`.

Run pytest:

```bash
pytest
```

```text
============================= test session starts ==============================
collected 2 items

tests/test_core.py::test_successful_payment PASSED                       [ 50%]
tests/test_database.py::test_save_and_retrieve_transaction PASSED        [100%]

============================== 2 passed in 0.02s ===============================
```

Works identically to `pytest.ini`.

### When to Use setup.cfg

**Use `setup.cfg` when**:
- You already have it for package configuration
- You want all configuration in one file
- You're using setuptools

**Avoid `setup.cfg` when**:
- You're using modern `pyproject.toml` for packaging
- You want pytest-specific configuration to be obvious

## Iteration 3: Using pyproject.toml (Modern Standard)

`pyproject.toml` is the modern Python standard (PEP 518). It uses TOML format, which is more readable than INI:

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "payflow"
version = "1.0.0"
dependencies = [
    "requests>=2.31.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.3",
    "pytest-cov>=4.1.0",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
addopts = [
    "-v",
    "--tb=short",
    "--strict-markers",
]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks tests as integration tests",
    "unit: marks tests as unit tests",
]
```

**Key differences from INI format**:
- Section is `[tool.pytest.ini_options]`
- Lists use array syntax: `["item1", "item2"]`
- Strings are quoted
- More structured and readable

Run pytest:

```bash
pytest
```

```text
============================= test session starts ==============================
collected 2 items

tests/test_core.py::test_successful_payment PASSED                       [ 50%]
tests/test_database.py::test_save_and_retrieve_transaction PASSED        [100%]

============================== 2 passed in 0.02s ===============================
```

Identical behavior, cleaner syntax.

## Iteration 4: Demonstrating Configuration in Action

Let's add a marker to a test and see configuration enforcement:

```python
# tests/test_core.py
import pytest
from payflow.core import process_payment

@pytest.mark.slow
def test_successful_payment():
    result = process_payment(100.00, "USD")
    assert result["status"] == "completed"

@pytest.mark.typo_marker  # Intentional typo
def test_invalid_currency():
    with pytest.raises(ValueError):
        process_payment(100.00, "INVALID")
```

Run pytest:

```bash
pytest
```

```text
============================= test session starts ==============================
collected 2 items

ERROR: Unknown pytest.mark.typo_marker - is this a typo?  You can register custom marks to avoid this warning - for details, see https://docs.pytest.org/en/stable/how-to/mark.html
```

### Diagnostic Analysis: Configuration Enforcement

**What happened?**

The `--strict-markers` option (from our configuration) caught the typo. Without it, pytest would silently ignore the unknown marker.

**The error tells us**:
1. **Unknown marker detected**: `typo_marker` isn't registered
2. **Suggestion**: Register it in configuration
3. **Documentation link**: How to fix it

**Fix the typo**:

```python
# tests/test_core.py
@pytest.mark.unit  # Fixed: using registered marker
def test_invalid_currency():
    with pytest.raises(ValueError):
        process_payment(100.00, "INVALID")
```

Now it works:

```bash
pytest
```

```text
============================= test session starts ==============================
collected 2 items

tests/test_core.py::test_successful_payment PASSED                       [ 50%]
tests/test_core.py::test_invalid_currency PASSED                         [100%]

============================== 2 passed in 0.02s ===============================
```

## Configuration Priority: Which File Wins?

If multiple configuration files exist, pytest uses this priority order:

1. **`pytest.ini`** (highest priority)
2. **`pyproject.toml`**
3. **`tox.ini`**
4. **`setup.cfg`** (lowest priority)

**Recommendation**: Use only one configuration file to avoid confusion.

## The Complete Configuration: Production-Ready

Here's a comprehensive `pyproject.toml` for a real project:

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "payflow"
version = "1.0.0"
description = "Payment processing library"
requires-python = ">=3.8"
dependencies = [
    "requests>=2.31.0",
    "sqlalchemy>=2.0.23",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.4.3",
    "pytest-cov>=4.1.0",
    "pytest-asyncio>=0.21.0",
    "black>=23.11.0",
    "mypy>=1.7.1",
]

[tool.pytest.ini_options]
# Test discovery
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]

# Output and behavior
addopts = [
    "-v",                      # Verbose output
    "--tb=short",              # Shorter tracebacks
    "--strict-markers",        # Fail on unknown markers
    "--strict-config",         # Fail on config errors
    "-ra",                     # Show summary of all test outcomes
    "--cov=payflow",          # Coverage for payflow package
    "--cov-report=term-missing",  # Show missing lines
    "--cov-report=html",      # Generate HTML coverage report
]

# Markers
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: marks tests requiring external services",
    "unit: marks tests as unit tests",
    "asyncio: marks tests as async",
]

# Asyncio configuration
asyncio_mode = "auto"

# Minimum coverage threshold
[tool.coverage.run]
source = ["payflow"]
omit = ["tests/*"]

[tool.coverage.report]
fail_under = 80
show_missing = true
```

## Decision Framework: Which Configuration File?

| Scenario | Recommended File | Rationale |
|----------|-----------------|-----------|
| New project (2024+) | `pyproject.toml` | Modern standard, cleaner syntax |
| Existing project with `setup.cfg` | `setup.cfg` | Consistency with existing config |
| Pytest-only configuration | `pytest.ini` | Explicit, pytest-specific |
| Legacy project | `setup.cfg` or `pytest.ini` | Depends on existing tooling |

## Common Configuration Options Reference

### Test Discovery

```toml
[tool.pytest.ini_options]
testpaths = ["tests"]              # Where to look for tests
python_files = ["test_*.py"]       # Test file patterns
python_classes = ["Test*"]         # Test class patterns
python_functions = ["test_*"]      # Test function patterns
```

### Output Control

```toml
[tool.pytest.ini_options]
addopts = [
    "-v",              # Verbose: show test names
    "-vv",             # Extra verbose: show more details
    "-q",              # Quiet: minimal output
    "--tb=short",      # Short tracebacks
    "--tb=long",       # Long tracebacks (default)
    "--tb=no",         # No tracebacks
    "-ra",             # Show summary of all outcomes
    "-rA",             # Show summary with passed tests too
]
```

### Marker Configuration

```toml
[tool.pytest.ini_options]
markers = [
    "slow: marks tests as slow",
    "integration: integration tests",
    "unit: unit tests",
]
addopts = ["--strict-markers"]  # Enforce marker registration
```

### Coverage Integration

```toml
[tool.pytest.ini_options]
addopts = [
    "--cov=mypackage",           # Measure coverage
    "--cov-report=term-missing", # Show missing lines
    "--cov-report=html",         # Generate HTML report
    "--cov-fail-under=80",       # Fail if coverage < 80%
]
```

## Common Failure Modes

### Symptom: Configuration not being applied

**Diagnostic clues**:
- Options in config file don't take effect
- Pytest seems to ignore the configuration

**Root cause**: Multiple configuration files exist, and pytest is using a different one

**Solution**: Check which config file pytest is using:

```bash
pytest --version -v
```

```text
pytest 7.4.3
configfile: /home/user/payflow_project/pytest.ini
```

Delete or rename conflicting configuration files.

### Symptom: "ERROR: Unknown pytest.mark.X"

**Pytest output**:

```text
ERROR: Unknown pytest.mark.integration - is this a typo?
```

**Root cause**: Marker used but not registered, and `--strict-markers` is enabled

**Solution**: Add marker to configuration:

```toml
[tool.pytest.ini_options]
markers = [
    "integration: marks tests as integration tests",
]
```

### Symptom: Configuration syntax error

**Pytest output**:

```text
ERROR: configuration error: could not load pyproject.toml
```

**Root cause**: Invalid TOML syntax

**Solution**: Validate TOML syntax. Common issues:
- Missing quotes around strings
- Incorrect array syntax
- Mismatched brackets

Use a TOML validator or IDE with TOML support.

## Key Principles

1. **Use one configuration file**: Avoid conflicts between multiple files
2. **Prefer `pyproject.toml`**: Modern standard, cleaner syntax
3. **Register all markers**: Use `--strict-markers` to catch typos
4. **Document options**: Add comments explaining why each option exists
5. **Share configuration**: Commit config file to version control
6. **Start minimal**: Add options as needed, don't copy-paste everything

## Common Configuration Options

## Beyond the Basics: Powerful Configuration Patterns

You've seen basic configuration—test discovery, output formatting, markers. Now let's explore configuration options that solve real problems in professional projects.

We'll continue with our `payflow` project and progressively add configuration as we encounter specific challenges.

## The Reference Scenario: A Growing Test Suite

Our `payflow` project now has:
- 50+ unit tests (fast, isolated)
- 20+ integration tests (slower, use database)
- 10+ end-to-end tests (slowest, full system)

Running all tests takes 2 minutes. During development, we want fast feedback. Let's configure pytest to handle this.

## Iteration 1: Controlling Test Output Verbosity

### The Problem: Too Much or Too Little Information

Run tests with default output:

```bash
pytest
```

```text
============================= test session starts ==============================
collected 80 items

tests/unit/test_core.py ........                                         [ 10%]
tests/unit/test_database.py .....                                        [ 16%]
tests/integration/test_payment_flow.py ....                              [ 21%]
...
============================== 80 passed in 45.23s =============================
```

**Problem**: When a test fails, you want details. When all pass, you want minimal output.

### Solution: Conditional Verbosity

```toml
# pyproject.toml
[tool.pytest.ini_options]
addopts = [
    "-ra",              # Show summary of all test outcomes
    "--tb=short",       # Short tracebacks on failure
]
```

The `-ra` flag shows a summary of all outcomes:
- **r**: Show summary
- **a**: All outcomes (passed, failed, skipped, xfailed, etc.)

Run tests:

```bash
pytest
```

```text
============================= test session starts ==============================
collected 80 items

tests/unit/test_core.py ........                                         [ 10%]
...
============================== 80 passed in 45.23s =============================
========================= short test summary info ==============================
PASSED tests/unit/test_core.py::test_successful_payment
PASSED tests/unit/test_core.py::test_invalid_currency
...
```

Now you see which tests passed without verbose output during execution.

### Advanced: Show Only Failures

```toml
[tool.pytest.ini_options]
addopts = [
    "-rfE",             # Show only failed and error tests
    "--tb=short",
]
```

- **r**: Show summary
- **f**: Failed tests
- **E**: Error tests

When all tests pass, output is minimal. When tests fail, you see exactly which ones.

## Iteration 2: Filtering Tests by Duration

### The Problem: Slow Tests Block Fast Feedback

Some tests are slow by nature (database operations, API calls). During development, you want to run only fast tests.

First, mark slow tests:

```python
# tests/integration/test_payment_flow.py
import pytest
from payflow.core import process_payment
from payflow.database import TransactionDB

@pytest.mark.slow
def test_payment_with_database_commit():
    """This test is slow because it commits to a real database."""
    db = TransactionDB()
    result = process_payment(100.00)
    db.save_transaction(result["transaction_id"], result)
    db.commit()  # Slow operation
    assert db.get_transaction(result["transaction_id"])
```

Configure markers:

```toml
[tool.pytest.ini_options]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
]
```

Now run only fast tests:

```bash
pytest -m "not slow"
```

```text
============================= test session starts ==============================
collected 80 items / 10 deselected / 70 selected

tests/unit/test_core.py ........                                         [ 11%]
...
==================== 70 passed, 10 deselected in 5.23s =========================
```

**Result**: Tests run in 5 seconds instead of 45 seconds.

### Making This the Default

Add to configuration:

```toml
[tool.pytest.ini_options]
addopts = [
    "-m", "not slow",   # Skip slow tests by default
]
```

Now `pytest` runs fast tests by default. To run all tests:

```bash
pytest -m ""  # Empty marker expression = run all
```

## Iteration 3: Configuring Test Timeouts

### The Problem: Hanging Tests

A test with an infinite loop or deadlock hangs forever:

```python
# tests/unit/test_core.py
def test_infinite_loop():
    while True:  # Oops!
        pass
```

Run pytest:

```bash
pytest tests/unit/test_core.py::test_infinite_loop
```

**Result**: Pytest hangs. You must manually kill it (Ctrl+C).

### Solution: pytest-timeout Plugin

Install the plugin:

```bash
pip install pytest-timeout
```

Configure a global timeout:

```toml
[tool.pytest.ini_options]
timeout = 10  # All tests must complete within 10 seconds
```

Run the hanging test:

```bash
pytest tests/unit/test_core.py::test_infinite_loop
```

```text
============================= test session starts ==============================
collected 1 item

tests/unit/test_core.py::test_infinite_loop FAILED                       [100%]

================================== FAILURES ====================================
__________________________ test_infinite_loop __________________________________
+++ Timeout +++
```

### Diagnostic Analysis: Timeout Failure

**The output tells us**:
1. **Test failed**: Not passed or skipped
2. **Reason**: `+++ Timeout +++`
3. **Duration**: Exceeded 10 seconds

**Root cause**: Infinite loop or deadlock

**Solution**: Fix the test logic or increase timeout for specific tests:

```python
@pytest.mark.timeout(30)  # This test needs 30 seconds
def test_slow_operation():
    # Long-running operation
    pass
```

## Iteration 4: Parallel Test Execution

### The Problem: Sequential Execution is Slow

Even fast tests take time when you have hundreds of them. Running sequentially wastes CPU cores.

Install pytest-xdist:

```bash
pip install pytest-xdist
```

Run tests in parallel:

```bash
pytest -n auto
```

The `-n auto` flag uses all available CPU cores.

**Result**: Tests run 4x faster on a 4-core machine.

### Making Parallel Execution the Default

```toml
[tool.pytest.ini_options]
addopts = [
    "-n", "auto",       # Parallel execution
]
```

**Warning**: Parallel execution requires tests to be independent. Tests that share state (files, database) may fail randomly.

### Disabling Parallel Execution for Specific Tests

Some tests can't run in parallel (e.g., tests that modify global state):

```python
@pytest.mark.serial  # Custom marker
def test_modifies_global_state():
    global_config.set("key", "value")
    assert global_config.get("key") == "value"
```

Configure pytest-xdist to respect this marker:

```toml
[tool.pytest.ini_options]
markers = [
    "serial: marks tests that must run serially",
]
```

Run serial tests separately:

```bash
pytest -m serial -n 0  # No parallelization
pytest -m "not serial" -n auto  # Parallelize the rest
```

## Iteration 5: Configuring Coverage Thresholds

### The Problem: Coverage Regression

Your project has 85% test coverage. A new developer adds code without tests, dropping coverage to 75%.

Configure coverage to fail below a threshold:

```toml
[tool.pytest.ini_options]
addopts = [
    "--cov=payflow",
    "--cov-report=term-missing",
    "--cov-fail-under=80",  # Fail if coverage < 80%
]
```

Add untested code:

```python
# payflow/core.py
def new_feature():
    """This function has no tests."""
    return "untested"
```

Run tests:

```bash
pytest
```

```text
============================= test session starts ==============================
collected 80 items

tests/unit/test_core.py ........                                         [ 10%]
...
============================== 80 passed in 5.23s ===============================

---------- coverage: platform linux, python 3.11.6 -----------
Name                    Stmts   Miss  Cover   Missing
-----------------------------------------------------
payflow/__init__.py         0      0   100%
payflow/core.py            45      5    89%   78-82
payflow/database.py        32      0   100%
-----------------------------------------------------
TOTAL                      77      5    94%

FAIL Required test coverage of 80% not reached. Total coverage: 75.32%
```

### Diagnostic Analysis: Coverage Failure

**The output tells us**:
1. **All tests passed**: 80 passed
2. **Coverage measured**: 75.32%
3. **Threshold not met**: Required 80%
4. **Missing lines**: Lines 78-82 in `core.py`

**Root cause**: New code added without tests

**Solution**: Write tests for the new feature or adjust the threshold temporarily.

## Iteration 6: Configuring Warning Filters

### The Problem: Deprecation Warnings Clutter Output

Third-party libraries emit deprecation warnings:

```bash
pytest
```

```text
============================= test session starts ==============================
collected 80 items

tests/unit/test_core.py ........
/usr/lib/python3.11/site-packages/requests/api.py:123: DeprecationWarning: 
  The 'verify' parameter is deprecated. Use 'ssl_context' instead.
...
============================== 80 passed in 5.23s ===============================
```

### Solution: Filter Warnings

```toml
[tool.pytest.ini_options]
filterwarnings = [
    "error",                                    # Treat warnings as errors
    "ignore::DeprecationWarning:requests.*",   # Ignore requests deprecations
    "ignore::PendingDeprecationWarning",       # Ignore pending deprecations
]
```

**Explanation**:
- `"error"`: Convert all warnings to errors (strict mode)
- `"ignore::DeprecationWarning:requests.*"`: Ignore deprecation warnings from the `requests` library
- `"ignore::PendingDeprecationWarning"`: Ignore pending deprecations globally

Run tests:

```bash
pytest
```

```text
============================= test session starts ==============================
collected 80 items

tests/unit/test_core.py ........                                         [ 10%]
...
============================== 80 passed in 5.23s ===============================
```

Clean output—no warnings.

## The Complete Configuration: Production-Ready

Here's a comprehensive configuration incorporating all patterns:

```toml
# pyproject.toml
[tool.pytest.ini_options]
# Test discovery
testpaths = ["tests"]
python_files = ["test_*.py"]
python_classes = ["Test*"]
python_functions = ["test_*"]

# Output control
addopts = [
    # Verbosity
    "-ra",                      # Show summary of all outcomes
    "--tb=short",               # Short tracebacks
    
    # Performance
    "-n", "auto",               # Parallel execution
    "--timeout=10",             # Global timeout
    
    # Coverage
    "--cov=payflow",
    "--cov-report=term-missing",
    "--cov-report=html",
    "--cov-fail-under=80",
    
    # Quality gates
    "--strict-markers",         # Fail on unknown markers
    "--strict-config",          # Fail on config errors
]

# Markers
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "integration: integration tests requiring external services",
    "unit: unit tests",
    "serial: tests that must run serially",
]

# Warning filters
filterwarnings = [
    "error",                                    # Warnings as errors
    "ignore::DeprecationWarning:requests.*",   # Ignore requests warnings
]

# Timeout configuration
timeout = 10
timeout_method = "thread"

# Coverage configuration
[tool.coverage.run]
source = ["payflow"]
omit = [
    "tests/*",
    "*/migrations/*",
]

[tool.coverage.report]
fail_under = 80
show_missing = true
skip_covered = false
```

## Configuration Options Reference

### Test Selection

| Option | Purpose | Example |
|--------|---------|---------|
| `testpaths` | Where to look for tests | `testpaths = ["tests"]` |
| `python_files` | Test file patterns | `python_files = ["test_*.py"]` |
| `python_classes` | Test class patterns | `python_classes = ["Test*"]` |
| `python_functions` | Test function patterns | `python_functions = ["test_*"]` |

### Output Control

| Option | Purpose | Example |
|--------|---------|---------|
| `-v` | Verbose output | Show test names |
| `-q` | Quiet output | Minimal output |
| `--tb=short` | Short tracebacks | Less noise on failure |
| `-ra` | Show all outcomes | Summary of all tests |
| `-rfE` | Show only failures | Minimal when passing |

### Performance

| Option | Purpose | Example |
|--------|---------|---------|
| `-n auto` | Parallel execution | Use all CPU cores |
| `--timeout=N` | Global timeout | Fail after N seconds |
| `-x` | Stop on first failure | Fast feedback |
| `--maxfail=N` | Stop after N failures | Limit failure cascade |

### Coverage

| Option | Purpose | Example |
|--------|---------|---------|
| `--cov=PKG` | Measure coverage | `--cov=payflow` |
| `--cov-report=term` | Terminal report | Show in console |
| `--cov-report=html` | HTML report | Generate HTML |
| `--cov-fail-under=N` | Minimum coverage | Fail if below N% |

### Quality Gates

| Option | Purpose | Example |
|--------|---------|---------|
| `--strict-markers` | Fail on unknown markers | Catch typos |
| `--strict-config` | Fail on config errors | Validate config |
| `--lf` | Run last failed | Rerun failures |
| `--ff` | Failed first | Prioritize failures |

## Decision Framework: Which Options to Use?

| Project Stage | Recommended Options | Rationale |
|--------------|---------------------|-----------|
| Early development | `-v`, `--tb=short` | Clear feedback |
| Growing test suite | `-n auto`, `-m "not slow"` | Fast feedback |
| CI/CD pipeline | `--cov-fail-under=80`, `--strict-markers` | Quality gates |
| Production-ready | All of the above | Comprehensive |

## Common Failure Modes

### Symptom: Tests pass locally but fail in CI

**Diagnostic clues**:
- Local: `pytest` passes
- CI: `pytest` fails with timeout

**Root cause**: Different timeout configuration

**Solution**: Ensure CI uses the same configuration file:

```yaml
# .github/workflows/test.yml
- name: Run tests
  run: pytest  # Uses pyproject.toml automatically
```

### Symptom: Parallel tests fail randomly

**Pytest output**:

```text
tests/integration/test_database.py::test_transaction FAILED
tests/integration/test_database.py::test_transaction PASSED  # Flaky!
```

**Root cause**: Tests share state (database, files)

**Solution**: Disable parallel execution for integration tests:

```bash
pytest tests/unit/ -n auto          # Parallel
pytest tests/integration/ -n 0      # Serial
```

## Key Principles

1. **Start minimal**: Add options as problems arise
2. **Document choices**: Comment why each option exists
3. **Test configuration**: Verify options work as expected
4. **Share configuration**: Commit to version control
5. **Use quality gates**: Enforce standards automatically
6. **Optimize for feedback speed**: Fast tests during development, comprehensive in CI

## Setting Up Your IDE for Testing

## The Problem: Manual Test Execution is Slow

You're developing a feature. Your workflow looks like this:

1. Write code
2. Switch to terminal
3. Type `pytest tests/test_feature.py::test_specific_case`
4. Read output
5. Switch back to editor
6. Repeat

This context switching kills productivity. What if you could run tests directly from your editor?

## The Solution: IDE Integration

Modern IDEs integrate pytest directly into the development environment. You can:
- Run tests with a keyboard shortcut
- See test results inline
- Debug failing tests with breakpoints
- Navigate to failing lines instantly

Let's set this up for the three most popular Python IDEs.

## Reference Project: Our payflow Test Suite

We'll use our `payflow` project with this structure:

```text
payflow_project/
├── payflow/
│   ├── __init__.py
│   ├── core.py
│   └── database.py
├── tests/
│   ├── unit/
│   │   ├── test_core.py
│   │   └── test_database.py
│   └── integration/
│       └── test_payment_flow.py
├── pyproject.toml
└── venv/
```

## Iteration 1: VS Code Setup

### Installing the Python Extension

1. Open VS Code
2. Press `Ctrl+Shift+X` (Extensions)
3. Search for "Python"
4. Install the official Microsoft Python extension

### Configuring Pytest Discovery

VS Code needs to know you're using pytest. Create `.vscode/settings.json`:

```json
{
    "python.testing.pytestEnabled": true,
    "python.testing.unittestEnabled": false,
    "python.testing.pytestArgs": [
        "tests"
    ],
    "python.testing.autoTestDiscoverOnSaveEnabled": true
}
```

**What this does**:
- `pytestEnabled`: Use pytest (not unittest)
- `pytestArgs`: Look for tests in `tests/` directory
- `autoTestDiscoverOnSaveEnabled`: Discover tests automatically when you save

### Discovering Tests

1. Open the Testing sidebar (flask icon on the left)
2. Click "Configure Python Tests"
3. Select "pytest"
4. Select "tests" as the test directory

VS Code scans your project and displays all tests in a tree view:

```text
TESTS
├── tests/
│   ├── unit/
│   │   ├── test_core.py
│   │   │   ├── test_successful_payment ✓
│   │   │   └── test_invalid_currency ✓
│   │   └── test_database.py
│   │       └── test_save_and_retrieve_transaction ✓
│   └── integration/
│       └── test_payment_flow.py
│           └── test_payment_is_saved_to_database ✓
```

### Running Tests from the IDE

**Run a single test**:
1. Click the play button next to the test name
2. Or right-click → "Run Test"

**Run all tests in a file**:
1. Click the play button next to the file name

**Run all tests**:
1. Click the play button at the top of the test tree

### Viewing Test Results

When tests run, VS Code shows results inline:

```python
# tests/unit/test_core.py
def test_successful_payment():  # ✓ Passed (0.01s)
    result = process_payment(100.00, "USD")
    assert result["status"] == "completed"

def test_invalid_currency():  # ✗ Failed (0.02s)
    with pytest.raises(ValueError):
        process_payment(100.00, "INVALID")
```

Failed tests show a red X. Click it to see the failure details in the terminal.

### Debugging Tests

Set a breakpoint in your test:
1. Click in the gutter next to a line number (red dot appears)
2. Right-click the test → "Debug Test"
3. Execution pauses at the breakpoint
4. Use the debug toolbar to step through code

Example debugging session:

```python
# tests/unit/test_core.py
def test_successful_payment():
    result = process_payment(100.00, "USD")  # ← Breakpoint here
    # Execution pauses, you can inspect 'result' in the debug console
    assert result["status"] == "completed"
```

### Keyboard Shortcuts

Add these to `.vscode/keybindings.json`:

```json
[
    {
        "key": "ctrl+shift+t",
        "command": "python.runCurrentTestFile"
    },
    {
        "key": "ctrl+shift+r",
        "command": "python.runTestAtCursor"
    }
]
```

Now:
- `Ctrl+Shift+T`: Run all tests in current file
- `Ctrl+Shift+R`: Run test under cursor

## Iteration 2: PyCharm Setup

PyCharm has built-in pytest support—no plugins needed.

### Configuring Pytest as the Default Test Runner

1. Open Settings (`Ctrl+Alt+S`)
2. Navigate to: Tools → Python Integrated Tools
3. Under "Testing", select "pytest" as the default test runner
4. Click "OK"

### Running Tests

**Run a single test**:
1. Right-click the test function
2. Select "Run 'pytest in test_core.py::test_successful_payment'"

**Run all tests in a file**:
1. Right-click the file in the project tree
2. Select "Run 'pytest in test_core.py'"

**Run all tests**:
1. Right-click the `tests/` directory
2. Select "Run 'pytest in tests'"

### Viewing Test Results

PyCharm shows a dedicated test runner window:

```text
Test Results
├── tests/unit/test_core.py
│   ├── ✓ test_successful_payment (0.01s)
│   └── ✗ test_invalid_currency (0.02s)
│       AssertionError: Expected ValueError but got None
│       Click to see full traceback →
```

Click a failed test to see:
- Full traceback
- Expected vs. actual values
- Link to the failing line

### Debugging Tests

1. Set a breakpoint (click in the gutter)
2. Right-click the test
3. Select "Debug 'pytest in test_core.py::test_successful_payment'"
4. Use the debugger toolbar to step through

### Running Tests with Coverage

1. Right-click the test/file/directory
2. Select "Run 'pytest in ...' with Coverage"
3. PyCharm shows coverage inline:

```python
# payflow/core.py
def process_payment(amount, currency="USD"):  # ✓ Covered
    if amount <= 0:                           # ✓ Covered
        raise ValueError("Amount must be positive")
    if currency not in ["USD", "EUR", "GBP"]: # ✓ Covered
        raise ValueError(f"Unsupported currency: {currency}")
    
    transaction_id = f"TXN-{amount}-{currency}"  # ✗ Not covered
    return {
        "transaction_id": transaction_id,
        "amount": amount,
        "currency": currency,
        "status": "completed"
    }
```

Green highlights = covered, red highlights = not covered.

### Keyboard Shortcuts

PyCharm's default shortcuts:
- `Ctrl+Shift+F10`: Run test at cursor
- `Shift+F10`: Rerun last test
- `Ctrl+Shift+F9`: Debug test at cursor
- `Shift+F9`: Debug last test

### Configuring Pytest Options

1. Open Run/Debug Configurations
2. Edit the pytest configuration
3. Add options in "Additional Arguments":

```text
-v --tb=short -m "not slow"
```

These options apply to all test runs from PyCharm.

## Iteration 3: Vim/Neovim Setup

For terminal-based editors, we'll use `vim-test` plugin.

### Installing vim-test

Add to your `.vimrc` or `init.vim`:

```vim
" Using vim-plug
Plug 'vim-test/vim-test'

" Configure test strategy
let test#strategy = "neovim"  " or "vimterminal" for Vim 8+
let test#python#runner = 'pytest'
```

Install plugins:

```vim
:PlugInstall
```

### Running Tests

**Run nearest test** (cursor on test function):

```vim
:TestNearest
```

**Run all tests in file**:

```vim
:TestFile
```

**Run all tests**:

```vim
:TestSuite
```

**Rerun last test**:

```vim
:TestLast
```

### Keyboard Mappings

Add to `.vimrc`:

```vim
" Test mappings
nmap <silent> <leader>tn :TestNearest<CR>
nmap <silent> <leader>tf :TestFile<CR>
nmap <silent> <leader>ts :TestSuite<CR>
nmap <silent> <leader>tl :TestLast<CR>
```

Now:
- `<leader>tn`: Run nearest test
- `<leader>tf`: Run file tests
- `<leader>ts`: Run all tests
- `<leader>tl`: Rerun last test

### Viewing Results in Split Window

Configure vim-test to show results in a split:

```vim
let test#strategy = "neovim"
let test#neovim#term_position = "vertical"  " Vertical split
```

Tests run in a vertical split, showing pytest output in real-time.

### Advanced: Async Test Execution

For Neovim with async support:

```vim
let test#strategy = "neovim"
let test#neovim#start_normal = 1  " Start in normal mode
```

Tests run asynchronously—you can continue editing while tests execute.

## Iteration 4: Configuring Test Discovery for All IDEs

### The Problem: IDE Can't Find Tests

Your IDE shows "No tests found" even though `pytest` works from the terminal.

### Diagnostic Analysis: Why Discovery Fails

**Common causes**:
1. **Wrong Python interpreter**: IDE using system Python, not virtual environment
2. **Wrong working directory**: IDE running from wrong location
3. **Missing pytest**: pytest not installed in the IDE's Python environment
4. **Configuration mismatch**: IDE configuration doesn't match pytest.ini

### Solution: Verify Python Interpreter

**VS Code**:
1. Press `Ctrl+Shift+P`
2. Type "Python: Select Interpreter"
3. Choose the interpreter from your `venv/` directory

**PyCharm**:
1. Open Settings → Project → Python Interpreter
2. Click the gear icon → Add
3. Select "Existing environment"
4. Navigate to `venv/bin/python`

**Vim/Neovim**:
Ensure you activate the virtual environment before starting Vim:

```bash
source venv/bin/activate
vim
```

### Solution: Verify Working Directory

**VS Code**: Check `.vscode/settings.json`:

```json
{
    "python.testing.cwd": "${workspaceFolder}"
}
```

**PyCharm**: Check Run/Debug Configuration:
- Working directory should be project root

### Solution: Install Pytest in IDE Environment

From the IDE's terminal:

```bash
pip install pytest
```

Verify:

```bash
python -m pytest --version
```

```text
pytest 7.4.3
```

## The Complete IDE Configuration

### VS Code: .vscode/settings.json

```json
{
    "python.testing.pytestEnabled": true,
    "python.testing.unittestEnabled": false,
    "python.testing.pytestArgs": [
        "tests",
        "-v",
        "--tb=short"
    ],
    "python.testing.autoTestDiscoverOnSaveEnabled": true,
    "python.testing.cwd": "${workspaceFolder}",
    "python.defaultInterpreterPath": "${workspaceFolder}/venv/bin/python"
}
```

### PyCharm: Run/Debug Configuration

1. Edit Configurations → Templates → Python tests → pytest
2. Set:
   - Target: Custom
   - Working directory: Project root
   - Additional Arguments: `-v --tb=short`
   - Python interpreter: Project venv

### Vim/Neovim: .vimrc

```vim
" vim-test configuration
let test#strategy = "neovim"
let test#python#runner = 'pytest'
let test#python#pytest#options = '-v --tb=short'

" Mappings
nmap <silent> <leader>tn :TestNearest<CR>
nmap <silent> <leader>tf :TestFile<CR>
nmap <silent> <leader>ts :TestSuite<CR>
nmap <silent> <leader>tl :TestLast<CR>
```

## Decision Framework: Which IDE Features to Use?

| Feature | When to Use | When to Skip |
|---------|-------------|--------------|
| Inline test results | Always | Never—essential feedback |
| Test tree view | Large test suites | Small projects (&lt;10 tests) |
| Debugging integration | Complex failures | Simple assertion failures |
| Coverage visualization | Improving coverage | Initial development |
| Auto-discovery | Always | Never—saves time |

## Common Failure Modes

### Symptom: "No tests found" in IDE but pytest works in terminal

**Diagnostic clues**:
- Terminal: `pytest` finds and runs tests
- IDE: Shows "No tests collected"

**Root cause**: IDE using different Python interpreter

**Solution**: Configure IDE to use project's virtual environment

### Symptom: Tests run but imports fail in IDE

**Error in IDE**:

```text
ModuleNotFoundError: No module named 'payflow'
```

**Root cause**: Package not installed in editable mode

**Solution**:

```bash
pip install -e .
```

### Symptom: IDE runs tests from wrong directory

**Pytest output**:

```text
ERROR: file not found: tests/test_core.py
```

**Root cause**: IDE's working directory is wrong

**Solution**: Configure working directory to project root in IDE settings

## Key Principles

1. **Use your IDE's test runner**: Faster than switching to terminal
2. **Configure the virtual environment**: Ensure IDE uses project's Python
3. **Set up keyboard shortcuts**: Minimize mouse usage
4. **Use inline results**: See pass/fail without switching windows
5. **Debug with breakpoints**: More efficient than print statements
6. **Run tests frequently**: IDE integration makes this effortless
