# Chapter 3: Setting Up Your Testing Environment

## Project Structure Best Practices

## The Foundation of a Maintainable Project

Before writing a single test, the most crucial decision you'll make is how to structure your project. A well-organized project is easy to navigate, test, and package. A disorganized one creates friction at every step, leading to confusing imports, difficulty in separating test code from application code, and deployment headaches.

This chapter is about building a solid foundation. We will start with a common but problematic structure and iteratively refine it, demonstrating *why* each improvement is necessary by showing the exact failure it prevents.

### Phase 1: Establish the Reference Implementation

Let's begin with a simple but realistic example that will be our anchor for this chapter. We'll build a `Wallet` application.

Here is the application code:

```python
# wallet.py

class InsufficientAmount(Exception):
    """Custom exception for when a wallet doesn't have enough funds."""
    pass

class Wallet:
    """A simple wallet to hold and manage a balance."""
    def __init__(self, initial_amount=0):
        if initial_amount < 0:
            raise ValueError("Initial amount cannot be negative.")
        self.balance = initial_amount

    def spend_cash(self, amount):
        if self.balance < amount:
            raise InsufficientAmount(f"You don't have {amount} to spend.")
        self.balance -= amount

    def add_cash(self, amount):
        self.balance += amount
```

And here is a basic test for it:

```python
# test_wallet.py

from wallet import Wallet, InsufficientAmount
import pytest

def test_default_initial_amount():
    wallet = Wallet()
    assert wallet.balance == 0

def test_setting_initial_amount():
    wallet = Wallet(100)
    assert wallet.balance == 100

def test_wallet_add_cash():
    wallet = Wallet(10)
    wallet.add_cash(90)
    assert wallet.balance == 100

def test_wallet_spend_cash():
    wallet = Wallet(20)
    wallet.spend_cash(10)
    assert wallet.balance == 10

def test_wallet_spend_cash_raises_exception_on_insufficient_amount():
    wallet = Wallet()
    with pytest.raises(InsufficientAmount):
        wallet.spend_cash(100)
```

### The "Flat Layout" Problem

The simplest way to organize these files is to put them all in the root directory. This is often called a "flat layout."

```bash
wallet-project/
├── wallet.py
└── test_wallet.py
```

Let's run pytest from the `wallet-project/` directory.

```bash
$ pytest
========================= test session starts ==========================
platform linux -- Python 3.11.5, pytest-8.1.1, pluggy-1.4.0
rootdir: /path/to/wallet-project
collected 5 items

test_wallet.py .....                                             [100%]

========================== 5 passed in 0.01s ===========================
```

It works! So, what's the problem? This structure seems simple and effective.

The "flat layout" is a trap. It works for tiny projects but breaks down quickly, creating subtle and frustrating problems as your project grows:

1.  **Ambiguous Imports:** Is `import wallet` referring to `wallet.py` in the current directory or an installed package named `wallet`? This ambiguity can cause your tests to pass locally but fail in production or CI/CD environments where the code is installed as a package.
2.  **Source vs. Test Confusion:** As the project grows, your root directory becomes a cluttered mix of application code, test files, configuration files, and documentation. It's hard to tell what's part of the deliverable application and what's for testing.
3.  **Packaging Nightmares:** When you try to package your application for distribution (e.g., to PyPI), it's difficult to configure the packaging tools to include *only* your application code (`wallet.py`) and exclude your tests (`test_wallet.py`).

To build a professional, scalable testing setup, we must first fix our project structure.

## Creating a tests/ Directory

## Iteration 1: Separating Tests from Source Code

Our first step is to create a clear separation between the code that runs the application and the code that tests it. The standard convention is to place all test files in a dedicated `tests/` directory.

### Current State Recap

Our project is a flat directory. Tests and source code are mixed together.

```bash
wallet-project/
├── wallet.py
└── test_wallet.py
```

### Current Limitation

This structure is messy and makes packaging difficult. We need to isolate the test code.

### New Scenario: Moving the Test File

Let's create a `tests/` directory and move our test file into it.

```bash
wallet-project/
├── wallet.py
└── tests/
    └── test_wallet.py
```

Now, let's try to run pytest again from the root `wallet-project/` directory.

```bash
$ pytest
```

### Failure Demonstration

This simple change breaks our tests completely.

```bash
========================= test session starts ==========================
platform linux -- Python 3.11.5, pytest-8.1.1, pluggy-1.4.0
rootdir: /path/to/wallet-project
collected 0 items / 1 error

================================ ERRORS ================================
___________________ ERROR collecting tests/test_wallet.py ____________________
ImportError while importing test module '/path/to/wallet-project/tests/test_wallet.py'.
Hint: make sure your test modules/packages have valid Python names.
Traceback:
/usr/lib/python3.11/importlib/__init__.py:126: in import_module
    return _bootstrap._gcd_import(name[level:], package, level)
tests/test_wallet.py:3: in <module>
    from wallet import Wallet, InsufficientAmount
E   ModuleNotFoundError: No module named 'wallet'
======================= short test summary info ========================
ERROR tests/test_wallet.py
!!!!!!!!!!!!!!!! Interrupted: 1 error during collection !!!!!!!!!!!!!!!!
========================== 1 error in 0.08s ============================
```

### Diagnostic Analysis: Reading the Failure

**The complete output**: The summary shows `1 error during collection`. This is a critical clue. The problem isn't a failing assertion; pytest couldn't even *load* the test file to find the tests.

**Let's parse this section by section**:

1.  **The summary line**: `ERROR collecting tests/test_wallet.py`
    -   **What this tells us**: The failure happened before any tests were run. Pytest was trying to discover tests inside `tests/test_wallet.py` and encountered a fatal error.

2.  **The traceback**:
    ```
    tests/test_wallet.py:3: in <module>
        from wallet import Wallet, InsufficientAmount
    E   ModuleNotFoundError: No module named 'wallet'
    ```
    -   **What this tells us**: The traceback points directly to the problem.
    -   **Key line**: `from wallet import Wallet, InsufficientAmount` followed by `ModuleNotFoundError: No module named 'wallet'`.

**Root cause identified**: Python can no longer find the `wallet` module.
**Why the current approach can't solve this**: When we run `pytest`, it adds the root directory (`wallet-project/`) to Python's import path (`sys.path`). The test runner looks inside `tests/test_wallet.py` and tries to execute `from wallet import ...`. Python looks for `wallet.py` in the current directory (`tests/`) and in the directories on `sys.path`. It can't find `wallet.py` inside `tests/`, and while `wallet-project/` is on the path, the `wallet.py` file is in the parent directory relative to the test file's new location. This creates an import problem.

**What we need**: We need a project structure that makes our application code a proper, installable package, so that imports are reliable and unambiguous, regardless of where the tests are located.

### The `src` Layout: The Professional Standard

The most robust solution is the "source" or `src` layout. This involves placing your main application code inside a `src` directory. This forces you to install your project to test it, which mimics how it will be used in the real world and eliminates a whole class of import problems.

**Technique Introduced**: The `src` layout and editable installs.

**Solution Implementation**:

1.  Create a `src` directory.
2.  Inside `src`, create a directory for your package (e.g., `wallet`).
3.  Move your application code (`wallet.py`) into this package directory.
4.  Create a `pyproject.toml` file to make your project installable.

Our new, much improved, project structure:

```bash
wallet-project/
├── pyproject.toml
├── src/
│   └── wallet/
│       ├── __init__.py   # Can be empty
│       └── wallet.py
└── tests/
    └── test_wallet.py
```

The `pyproject.toml` file tells Python's build tools that this is an installable project. A minimal version looks like this:

```toml
# pyproject.toml
[project]
name = "wallet"
version = "0.1.0"

[tool.setuptools.packages.find]
where = ["src"]
```

Now, the crucial step: we must **install our package in editable mode**. This creates a link from our virtual environment's `site-packages` directory to our source code. Any changes we make in `src/wallet/wallet.py` will be immediately available to our tests without reinstalling.

```bash
# First, ensure you are in a virtual environment (more on this next!)
# Then, from the root directory (wallet-project/):
pip install -e .
```

The `-e` stands for "editable". The `.` refers to the current directory, where `pyproject.toml` is located.

### Verification

With the project installed, let's run pytest again.

```bash
$ pytest
========================= test session starts ==========================
platform linux -- Python 3.11.5, pytest-8.1.1, pluggy-1.4.0
rootdir: /path/to/wallet-project
collected 5 items

tests/test_wallet.py .....                                       [100%]

========================== 5 passed in 0.01s ===========================
```

Success! The `ModuleNotFoundError` is gone. Because our `wallet` package is now properly installed, Python can find it no matter where the test runner is invoked from. We have achieved a clean separation of concerns.

### When to Apply This Solution

-   **What it optimizes for**: Reliability, maintainability, and preventing import errors. It ensures your test environment closely mirrors your production environment.
-   **What it sacrifices**: A tiny amount of initial setup complexity. You can no longer just run a Python script from a flat directory.
-   **When to choose this approach**: Always, for any project that you expect to last more than a day or be shared with others.
-   **When to avoid this approach**: Only for single-file, disposable scripts.

**Limitation preview**: Our project structure is now robust, but our development environment is not. We are likely installing packages into our global Python installation, which can lead to dependency conflicts between projects. We need to isolate our project's dependencies.

## Using Virtual Environments

## Iteration 2: Isolating Dependencies

Our project structure is solid, but it lives on a shaky foundation: the system-wide Python environment. Every `pip install` command we run without a virtual environment modifies our global collection of packages.

### Current State Recap

We have a `src` layout and have installed our package in editable mode.

```bash
wallet-project/
├── pyproject.toml
├── src/
│   └── wallet/
│       └── wallet.py
└── tests/
    └── test_wallet.py
```

### Current Limitation

Our project's dependencies (like `pytest`) are installed globally. This creates two major problems:

1.  **Dependency Hell**: Project A requires `pytest==7.0`, but Project B requires `pytest==8.0`. Installing one will break the other.
2.  **Implicit Dependencies**: If our code happens to use a package that's already installed globally (e.g., `requests`), we might forget to list it as a dependency for our project. The code works on our machine but will crash for anyone else who tries to run it on a clean system.

### New Scenario: Ensuring a Reproducible Environment

We need to guarantee that our project has its own isolated set of dependencies that don't interfere with other projects and are explicitly defined.

The "failure" here isn't a test crash, but a workflow failure. Imagine giving your project to a colleague. They run `pip install pytest` and get the latest version, which happens to be incompatible with your tests. Your test suite, which worked perfectly on your machine, now fails for them. This is a failure of reproducibility.

**Technique Introduced**: Python's built-in `venv` module for creating virtual environments.

A virtual environment is a self-contained directory tree that contains a Python installation for a particular version of Python, plus a number of additional packages.

### Solution Implementation

Let's create and activate a virtual environment for our project.

**Step 1: Create the virtual environment**

From the root of `wallet-project/`, run the following command. It's conventional to name the environment `venv` or `.venv`.

```bash
# This creates a directory named 'venv' containing a private Python installation.
python3 -m venv venv
```

You should also add `venv/` to your `.gitignore` file to prevent committing it to version control.

**Step 2: Activate the virtual environment**

Activation configures your current shell to use the Python interpreter and packages from within the `venv` directory.

-   On macOS and Linux:

```bash
source venv/bin/activate
```

-   On Windows (Command Prompt):

```bash
venv\Scripts\activate.bat
```

-   On Windows (PowerShell):

```bash
venv\Scripts\Activate.ps1
```

After activation, your shell prompt will typically change to show the name of the active environment, like `(venv) $`.

**Step 3: Install dependencies into the isolated environment**

Now that the environment is active, any `pip` command will operate *only* inside the `venv` directory. Let's install our project and its testing dependencies.

```bash
(venv) $ pip install --upgrade pip  # Good practice to use the latest pip
(venv) $ pip install pytest
(venv) $ pip install -e .           # Install our local 'wallet' package
```

If you run `pip list` now, you will see a very short list of packages—only what you just installed and their dependencies. This is our clean, isolated environment.

### Verification

Let's run our tests from within the activated virtual environment.

```bash
(venv) $ pytest
========================= test session starts ==========================
platform linux -- Python 3.11.5, pytest-8.1.1, pluggy-1.4.0
rootdir: /path/to/wallet-project
collected 5 items

tests/test_wallet.py .....                                       [100%]

========================== 5 passed in 0.01s ===========================
```

Everything still passes, but now we have a guarantee: our test run is completely self-contained and reproducible. Anyone else can create their own virtual environment, install the same dependencies, and get the exact same result.

**Limitation preview**: Our setup is now isolated and structured well. However, running tests still relies on remembering specific command-line options. If one developer runs `pytest` and another runs `pytest -v --strict-markers`, they might see different results or warnings. We need a way to enforce consistent test execution for everyone on the team.

## Pytest Configuration Files (pytest.ini, setup.cfg, pyproject.toml)

## Iteration 3: Centralizing Configuration

We have a professional project structure and an isolated environment. The final piece of the setup puzzle is creating a single source of truth for how our tests should be run. Relying on developers to remember command-line flags is a recipe for inconsistency.

### Current State Recap

We run `pytest` from within an activated virtual environment. The command is simple, but it uses pytest's default settings.

### Current Limitation

To get more informative output or enforce stricter checks, we need to add flags. For example:

-   `pytest -v`: for verbose output, showing one line per test.
-   `pytest --strict-markers`: to fail the test suite if an unregistered marker is used.

Forgetting these flags can lead to developers missing important information or introducing errors (like marker typos) that go unnoticed.

### New Scenario: Enforcing Consistent Test Runs

We want to ensure that every time anyone runs `pytest`, it automatically uses a standard set of options, like `-v`.

**Failure Demonstration**: The "failure" is the subtle difference in output and behavior.

Run without flags:

```bash
(venv) $ pytest
========================= test session starts ==========================
...
tests/test_wallet.py .....                                       [100%]
========================== 5 passed in 0.01s ===========================
```

Now run with the verbose flag:

```bash
(venv) $ pytest -v
========================= test session starts ==========================
...
tests/test_wallet.py::test_default_initial_amount PASSED         [ 20%]
tests/test_wallet.py::test_setting_initial_amount PASSED         [ 40%]
tests/test_wallet.py::test_wallet_add_cash PASSED                [ 60%]
tests/test_wallet.py::test_wallet_spend_cash PASSED              [ 80%]
tests/test_wallet.py::test_wallet_spend_cash_raises_exception_on_insufficient_amount PASSED [100%]
========================== 5 passed in 0.01s ===========================
```

The verbose output is much more informative. We want this to be the default, without having to type it every time.

**Technique Introduced**: Pytest configuration files.

Pytest automatically discovers and reads configuration from one of several files in your project's root directory. The search order is: `pytest.ini`, `pyproject.toml`, `tox.ini`, `setup.cfg`.

The modern standard is `pyproject.toml`, as it is the unified configuration file for all Python tooling (build systems, linters, formatters, and test runners). We will use it.

### Solution Implementation

We will add a new section to our existing `pyproject.toml` file to hold our pytest configuration.

**Before**:

```toml
# pyproject.toml
[project]
name = "wallet"
version = "0.1.0"

[tool.setuptools.packages.find]
where = ["src"]
```

**After**:

```toml
# pyproject.toml
[project]
name = "wallet"
version = "0.1.0"

[tool.setuptools.packages.find]
where = ["src"]

[tool.pytest.ini_options]
# Add command-line options here
addopts = "-v --strict-markers"
```

The `[tool.pytest.ini_options]` table is where all pytest settings go. The `addopts` key is a string of command-line arguments that pytest will automatically prepend to every run.

### Verification

Now, let's run the plain `pytest` command again, with no extra arguments.

```bash
(venv) $ pytest
========================= test session starts ==========================
...
tests/test_wallet.py::test_default_initial_amount PASSED         [ 20%]
tests/test_wallet.py::test_setting_initial_amount PASSED         [ 40%]
tests/test_wallet.py::test_wallet_add_cash PASSED                [ 60%]
tests/test_wallet.py::test_wallet_spend_cash PASSED              [ 80%]
tests/test_wallet.py::test_wallet_spend_cash_raises_exception_on_insufficient_amount PASSED [100%]
========================== 5 passed in 0.01s ===========================
```

Perfect. The output is automatically verbose. We have successfully centralized our testing configuration, ensuring every test run is consistent.

## Common Configuration Options

## Configuring Pytest for Professional Workflows

The `[tool.pytest.ini_options]` section in your `pyproject.toml` is your command center for controlling pytest's behavior. Let's explore some of the most valuable options you'll use in day-to-day development.

Our starting configuration:

```toml
# pyproject.toml

[tool.pytest.ini_options]
addopts = "-v --strict-markers"
```

### `testpaths`: Specifying Where to Find Tests

By default, pytest searches the entire project directory for tests. In a large project, this can be slow. You can speed up test discovery by telling pytest exactly where to look.

**Problem**: Slow test collection in large repositories with many files.
**Solution**: Use `testpaths` to limit the search.

```toml
[tool.pytest.ini_options]
addopts = "-v --strict-markers"
testpaths = ["tests"]
```

With this setting, `pytest` will *only* look for tests inside the `tests/` directory and ignore everything else.

### `minversion`: Enforcing a Pytest Version

**Problem**: A new developer joins the team and installs an old, incompatible version of pytest, causing strange errors.
**Solution**: Specify a minimum required version.

```toml
[tool.pytest.ini_options]
minversion = "8.0" # Require pytest version 8.0 or newer
addopts = "-v --strict-markers"
testpaths = ["tests"]
```

If someone tries to run the suite with an older version, pytest will exit immediately with a clear error message.

### `markers`: Registering Custom Markers

As we'll see in Chapter 6, markers are a powerful way to categorize tests (e.g., `@pytest.mark.slow`, `@pytest.mark.api`). The `--strict-markers` option in `addopts` will cause an error if you use a marker that hasn't been officially registered. This is a good thing—it prevents typos!

**Problem**: Typos in marker names (`@pytest.mark.slwo`) go unnoticed, and tests are not selected or skipped as intended.
**Solution**: Register all valid markers in the configuration file.

```toml
[tool.pytest.ini_options]
minversion = "8.0"
addopts = "-v --strict-markers"
testpaths = ["tests"]
markers = [
    "slow: marks tests as slow (select with '-m slow')",
    "api: marks tests that hit a live API endpoint",
]
```

Now, if you accidentally type `@pytest.mark.slwo`, the test suite will fail with an error telling you that the marker is not registered.

### Customizing Test Discovery

Pytest's discovery rules are sensible by default (it looks for `test_*.py` or `*_test.py` files), but you can override them if your project has different conventions.

**Problem**: Your project uses a different naming convention for test files, like `check_*.py`.
**Solution**: Use `python_files`, `python_classes`, and `python_functions` to change the discovery patterns.

```toml
[tool.pytest.ini_options]
# ... other options
python_files = "test_*.py *_test.py check_*.py"
python_classes = "Test* Check*"
python_functions = "test_* check_*"
```

This tells pytest to also discover tests in files like `check_wallet.py`, inside classes like `CheckWallet`, and from functions like `check_initial_balance`.

### A Complete Configuration Example

Here is what a robust, professional `pytest.ini_options` section might look like, combining these common settings.

```toml
# pyproject.toml

# ... [project] and [tool.setuptools.packages.find] sections ...

[tool.pytest.ini_options]
# Enforce a minimum pytest version for consistency
minversion = "8.0"

# Specify the directory where tests are located
testpaths = ["tests"]

# Add command-line options that should always be used
# -v: verbose
# -rA: show extra test summary info for all outcomes
# --strict-markers: fail on unregistered markers
# --color=yes: force color output (good for CI)
addopts = "-v -rA --strict-markers --color=yes"

# Register custom markers to avoid typos and provide help
markers = [
    "slow: marks tests as slow to run",
    "smoke: marks a small subset of critical-path tests",
    "regression: marks regression tests for a specific bug",
]
```
