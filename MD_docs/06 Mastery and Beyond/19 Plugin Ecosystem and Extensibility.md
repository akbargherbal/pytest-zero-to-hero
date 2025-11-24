# Chapter 19: Plugin Ecosystem and Extensibility

## Popular Pytest Plugins (pytest-html, pytest-xdist, pytest-sugar)

## The Power of Extensibility

Pytest's core is powerful, but its true strength lies in its extensibility. The plugin system allows developers to alter or extend nearly every aspect of pytest's behavior, from test collection and execution to reporting. This has fostered a rich ecosystem of third-party plugins that can solve common testing problems with a simple `pip install`.

In this chapter, we'll explore this ecosystem. We'll start by using some of the most popular plugins to solve real-world problems, then we'll dive into the mechanics of how plugins work, and finally, we'll build and package our own custom plugin.

### Phase 1: Establish the Reference Implementation

To see the value of plugins, we need a test suite with some tangible challenges. Let's create a simple data validation utility. Its job is to check data files for common issues. To simulate a real-world scenario, our tests will be intentionally a bit slow, making them a perfect candidate for optimization.

Our anchor example will be a `DataValidator` that performs checks on CSV-like data.

Here is the code for our utility:

```python
# project/validator.py
import time
import random

class DataValidationError(Exception):
    pass

class DataValidator:
    def __init__(self, data_source):
        # In a real app, this would read from a file or database
        self.data = data_source.splitlines()
        if not self.data:
            raise DataValidationError("Data source is empty")
        self.headers = self.data[0].split(',')
        self.rows = [line.split(',') for line in self.data[1:]]

    def check_row_length(self):
        """Simulates a check that all rows have the same number of columns as the header."""
        time.sleep(0.1) # Simulate I/O and processing
        header_len = len(self.headers)
        for i, row in enumerate(self.rows):
            if len(row) != header_len:
                raise DataValidationError(f"Row {i+1} has incorrect length")
        return True

    def check_unique_id(self, column_name):
        """Simulates a check for unique values in a given column."""
        time.sleep(0.15) # Simulate a more expensive check
        try:
            idx = self.headers.index(column_name)
        except ValueError:
            raise DataValidationError(f"Column '{column_name}' not found")
        
        seen_ids = set()
        for i, row in enumerate(self.rows):
            row_id = row[idx]
            if row_id in seen_ids:
                raise DataValidationError(f"Duplicate ID '{row_id}' found in column '{column_name}'")
            seen_ids.add(row_id)
        return True

    def check_value_range(self, column_name, min_val=0, max_val=100):
        """Simulates checking that values in a column are within a numeric range."""
        time.sleep(0.12) # Simulate another expensive check
        try:
            idx = self.headers.index(column_name)
        except ValueError:
            raise DataValidationError(f"Column '{column_name}' not found")

        for i, row in enumerate(self.rows):
            try:
                val = int(row[idx])
                if not (min_val <= val <= max_val):
                    raise DataValidationError(f"Value {val} in row {i+1} is out of range ({min_val}-{max_val})")
            except (ValueError, IndexError):
                raise DataValidationError(f"Invalid numeric value in row {i+1}, column '{column_name}'")
        return True
```

And here is our initial test suite. We'll create a fixture to provide valid sample data.

```python
# tests/test_validator.py
import pytest
from project.validator import DataValidator, DataValidationError

@pytest.fixture
def valid_data():
    """Provides a string of valid CSV data."""
    return (
        "id,name,value\n"
        "1,alpha,10\n"
        "2,beta,25\n"
        "3,gamma,99\n"
    )

def test_valid_data_passes_all_checks(valid_data):
    """A baseline test for valid data."""
    validator = DataValidator(valid_data)
    assert validator.check_row_length()
    assert validator.check_unique_id(column_name="id")
    assert validator.check_value_range(column_name="value", min_val=0, max_val=100)

def test_incorrect_row_length_fails(valid_data):
    """Tests that a row with a missing column raises an error."""
    invalid_data = valid_data.replace("25", "25,extra_col")
    validator = DataValidator(invalid_data)
    with pytest.raises(DataValidationError, match="incorrect length"):
        validator.check_row_length()

def test_duplicate_id_fails():
    """Tests that duplicate IDs are caught."""
    # This test creates its own data to be independent
    data = "id,name\n1,alpha\n1,beta"
    validator = DataValidator(data)
    with pytest.raises(DataValidationError, match="Duplicate ID '1'"):
        validator.check_unique_id(column_name="id")

def test_value_out_of_range_fails(valid_data):
    """Tests that a value outside the specified range fails."""
    invalid_data = valid_data.replace("99", "101")
    validator = DataValidator(invalid_data)
    with pytest.raises(DataValidationError, match="out of range"):
        validator.check_value_range(column_name="value", min_val=0, max_val=100)
```

Let's run this suite and see the standard pytest output.

```bash
$ pytest
============================= test session starts ==============================
platform linux -- Python 3.10.12, pytest-7.4.0, pluggy-1.2.0
rootdir: /path/to/project
collected 4 items

tests/test_validator.py ....                                             [100%]

============================== 4 passed in 1.51s ===============================
```

Our test suite works, but it has three problems that plugins can solve:
1.  **Readability**: The output is functional, but for larger suites, the simple dots (`....`) aren't very engaging or informative.
2.  **Reporting**: If we need to share these results, terminal output isn't a professional or persistent format.
3.  **Speed**: The suite took over 1.5 seconds for just four simple tests. As we add hundreds more, this will become a major bottleneck.

Let's tackle these one by one using popular plugins.

### Iteration 1: Improving Readability with `pytest-sugar`

**Current Limitation**: The default pytest output is minimal. It doesn't show test names as they run, and the progress indicator is just a series of dots.

`pytest-sugar` is a plugin that provides a much nicer user interface in the terminal, including a progress bar and instant feedback on failing tests.

First, let's install it.

```bash
pip install pytest-sugar
```

Now, let's run our tests again with no other changes. `pytest-sugar` automatically activates itself upon installation.

```bash
$ pytest

―――――――――――――――――――――――――――――― test session starts ―――――――――――――――――――――――――――――――
platform linux -- Python 3.10.12, pytest-7.4.0, pluggy-1.2.0
rootdir: /path/to/project
plugins: sugar-1.0.0
collected 4 items

 tests/test_validator.py ✓✓✓✓                                           100% ██████████

――――――――――――――――――――――――――――――――――― summary ――――――――――――――――――――――――――――――――――――
✅ 4 passed in 1.52s
```

**Expected vs. Actual Improvement**: The output is immediately clearer. We get a progress bar, and the checkmarks (`✓`) give a more positive sense of progress than the dots. If a test were to fail, `pytest-sugar` would print the failure immediately instead of waiting for the entire run to finish, providing faster feedback. This simple plugin enhances the developer experience with zero configuration.

### Iteration 2: Generating Reports with `pytest-html`

**Current Limitation**: Our test results vanish as soon as we close the terminal. We have no artifact to archive, email, or post on a dashboard.

`pytest-html` solves this by generating a self-contained HTML report with the results of the test run.

First, install it.

```bash
pip install pytest-html
```

To use it, we run pytest with an extra command-line option, `--html`, specifying the output file.

```bash
pytest --html=report.html --self-contained-html
```

The run looks the same in the terminal (you'll still see the `pytest-sugar` output), but now you'll have a new file, `report.html`, in your directory. Opening it in a browser shows a detailed, professional report with information about your environment, a summary of results, and a detailed breakdown of each test, including its duration.

This artifact is invaluable for continuous integration (CI) systems, quality assurance teams, and project managers.

### Iteration 3: Speeding Up the Suite with `pytest-xdist`

**Current Limitation**: Our tests run sequentially, one after another. The total time is the sum of all individual test times. Our four tests took ~1.5 seconds. A suite with 400 such tests would take over 2 minutes.

`pytest-xdist` allows you to run tests in parallel, distributing them across multiple CPU cores. Since our tests are independent of each other, they are perfect candidates for parallelization.

First, install it.

```bash
pip install pytest-xdist
```

To activate it, we use the `-n` flag to specify the number of parallel processes. A common choice is `auto`, which tells `pytest-xdist` to use the number of available CPU cores.

```bash
# First, let's get a baseline time without xdist
$ time pytest -q
....
4 passed in 1.51s

real    0m1.623s
user    0m0.315s
sys     0m0.048s

# Now, let's run with xdist
$ time pytest -q -n auto
....
4 passed in 0.55s

real    0m0.897s
user    0m0.641s
sys     0m0.102s
```

**Expected vs. Actual Improvement**: The results are dramatic. The test session duration reported by pytest dropped from **1.51s** to **0.55s**—nearly a 3x speedup on a machine with 4 cores. The `real` time reported by the `time` command shows a similar improvement.

`pytest-xdist` intelligently sends each test file to a different worker process. Because our tests are I/O-bound (due to `time.sleep`), parallelization allows the CPU to switch between them effectively, finishing the entire suite much faster than a sequential run. For large, slow test suites (e.g., those involving database access or network requests), `pytest-xdist` is an essential tool.

## Installing and Configuring Plugins

## Managing Your Plugins

As we saw in the previous section, using a plugin is typically a two-step process:
1.  Install the package using `pip`.
2.  Activate its functionality, either automatically or via a command-line flag.

### Installation

Plugins are standard Python packages, so they are installed with `pip`:

```bash
pip install pytest-xdist
```

It's crucial to manage these dependencies just like any other project dependency. You should add them to your `requirements.txt` or `pyproject.toml` file to ensure that your test environment is reproducible.

```text
# requirements.txt
pytest
pytest-sugar
pytest-html
pytest-xdist
```

### Discovering Installed Plugins

Pytest automatically discovers any installed packages that register as plugins. You can see which plugins are active in your environment by looking at the header of any pytest run:

```bash
$ pytest --version
pytest 7.4.0, pluggy 1.2.0
plugins: sugar-1.0.0, html-4.0.1, xdist-3.3.1
```

For even more detail, you can use the `--trace-config` flag, which shows where each plugin was loaded from. This is extremely useful for debugging issues with plugin loading.

```bash
$ pytest --trace-config
============================= test session starts ==============================
# ... (lots of output)
PLUGIN registered: <module 'sugar.plugin' from '/.../lib/python3.10/site-packages/sugar/plugin.py'>
PLUGIN registered: <module 'pytest_html.plugin' from '/.../lib/python3.10/site-packages/pytest_html/plugin.py'>
PLUGIN registered: <module 'pytest_xdist.plugin' from '/.../lib/python3.10/site-packages/pytest_xdist/plugin.py'>
# ... (more output)
```

### Configuration

Many plugins offer configuration options to customize their behavior. These are typically managed in your `pytest.ini`, `pyproject.toml`, or `setup.cfg` file, under a section named `[pytest]`.

For example, you can set command-line options by default using `addopts`. Instead of typing `pytest -n auto --html=report.html` every time, you can configure it like this:

**`pytest.ini`**

```ini
[pytest]
addopts = -n auto --html=report.html --self-contained-html
```

Now, simply running `pytest` will execute with those options automatically.

Plugins may also define their own custom configuration keys. For example, `pytest-html` allows you to customize the report title.

**`pytest.ini`**

```ini
[pytest]
addopts = -n auto --html=report.html --self-contained-html

[pytest-html]
report_title = My Project Test Report
```

Always consult the documentation for the specific plugin you are using to see the available configuration options.

## Creating Custom Plugins

## Extending Pytest Yourself

Using third-party plugins is powerful, but the real magic happens when you realize you can write your own. This allows you to create project-specific helpers, enforce custom conventions, or integrate with internal tools.

The simplest way to create a plugin is to place code in a `conftest.py` file at your project's root. Pytest automatically discovers and loads this file, making any hooks or fixtures defined within it available to your entire test suite.

### Iteration 4: Creating a Custom Test Timer Plugin

**Current Limitation**: We know the total test suite duration, but we don't have an easy way to see the duration of each individual test or to flag tests that are unusually slow. While `pytest --durations` exists, building our own simple version is the best way to learn the plugin mechanism.

**Goal**: We want to create a plugin that:
1.  Times each test individually.
2.  At the end of the run, prints a report of the 5 slowest tests.

To do this, we need to tap into pytest's execution process. We'll use **hooks**, which are special functions that pytest calls at specific points during its lifecycle. We'll place our hook implementations in `conftest.py`.

Here is the implementation for our simple timing plugin:

```python
# tests/conftest.py
import time

def pytest_runtest_setup(item):
    """Hook called before a test item is run."""
    item.start_time = time.time()

def pytest_runtest_teardown(item, nextitem):
    """Hook called after a test item is run."""
    item.end_time = time.time()
    item.duration = item.end_time - item.start_time

def pytest_sessionfinish(session):
    """Hook called after the entire test session finishes."""
    reporter = session.config.pluginmanager.get_plugin('terminalreporter')
    reporter.write_sep("=", "CUSTOM TEST DURATIONS REPORT")
    
    # Collect all items and their durations
    all_items = [item for item in session.items if hasattr(item, 'duration')]
    
    # Sort by duration, descending
    slowest_tests = sorted(all_items, key=lambda x: x.duration, reverse=True)
    
    # Report the top 5 slowest
    reporter.write_line("\nTop 5 slowest tests:")
    for item in slowest_tests[:5]:
        reporter.write_line(f"{item.nodeid}: {item.duration:.4f}s")
```

### Banish Magic with Mechanics: How This Works

Let's break down what's happening here. We've defined three functions with special names that pytest recognizes as hooks:

1.  **`pytest_runtest_setup(item)`**: Pytest calls this hook just before it executes a test function. The `item` object represents the test being run (e.g., `test_valid_data_passes_all_checks`). We attach a `start_time` attribute directly to this object to store the current time.

2.  **`pytest_runtest_teardown(item, nextitem)`**: This hook is called after the test has finished (and after any teardown fixtures). We record the `end_time` and calculate the `duration`, again storing it on the `item` object.

3.  **`pytest_sessionfinish(session)`**: Pytest calls this hook once at the very end of the entire test session. This is the perfect place to generate our summary report.
    - We get the `terminalreporter` object from the session, which allows us to print nicely formatted output to the console.
    - We collect all test items from the session.
    - We sort them by the `duration` attribute we added.
    - We print the top 5 slowest tests.

Now, let's run pytest. We don't need any special flags; the plugin in `conftest.py` is loaded automatically.

```bash
$ pytest
# ... (normal test output from pytest-sugar) ...

――――――――――――――――――――――――――――――――――― summary ――――――――――――――――――――――――――――――――――――
✅ 4 passed in 1.53s

======================= CUSTOM TEST DURATIONS REPORT =======================

Top 5 slowest tests:
tests/test_validator.py::test_valid_data_passes_all_checks: 0.3789s
tests/test_validator.py::test_duplicate_id_fails: 0.1534s
tests/test_validator.py::test_value_out_of_range_fails: 0.1227s
tests/test_validator.py::test_incorrect_row_length_fails: 0.1019s
```

**Expected vs. Actual Improvement**: It worked perfectly! After the standard summary, our custom report is printed, showing the exact duration of each test, sorted from slowest to fastest. We have successfully extended pytest's functionality to meet our specific needs. This demonstrates the core principle of plugins: you can attach your own logic to almost any part of the testing process.

## Hooks: How Pytest Plugins Work Under the Hood

## The Pytest Hook System

The functions we wrote in `conftest.py` (`pytest_runtest_setup`, etc.) are the heart of the plugin system. Pytest defines hundreds of these **hook specifications** that act as well-defined entry points for plugins to inject custom logic.

When pytest is about to perform an action (like collecting tests, running a test, or finishing the session), it checks if any loaded plugins (including `conftest.py`) have implemented the corresponding hook function. If so, it calls that function, passing in relevant context as arguments (like the `item` or `session` object).

This is a powerful design because it allows plugins to be modular and self-contained. A plugin only needs to implement the hooks it cares about.

### Key Hook Categories and Examples

While there are many hooks, they generally fall into a few main categories. Understanding these categories helps you know where to look when you want to build a plugin.

#### 1. Initialization and Configuration Hooks
These hooks run at the very beginning of a pytest session and are used for adding command-line options or modifying configuration.

-   **`pytest_addoption(parser)`**: Allows you to add custom command-line options. The `parser` object is used to define the option, its help text, and default value.
-   **`pytest_configure(config)`**: Called after command-line options have been parsed. This is where you can read your custom options and set up any global state for your plugin.

#### 2. Collection Hooks
These hooks are called when pytest is discovering test files and functions. They allow you to customize the collection process.

-   **`pytest_collection_modifyitems(session, config, items)`**: One of the most powerful hooks. It's called after all test items have been collected. The `items` argument is a list of all test functions pytest is about to run. You can reorder this list, remove items (deselect tests), or add custom markers. For example, you could write a plugin to automatically add a `@pytest.mark.slow` marker to any test whose name contains `_integration_`.

#### 3. Test Execution (Runtest) Hooks
This is the family of hooks we used for our timer plugin. They wrap the actual execution of a single test item.

-   **`pytest_runtest_protocol(item, nextitem)`**: This is the main hook that manages the execution of a single test. It's responsible for calling the setup, the test function itself, and the teardown.
-   **`pytest_runtest_setup(item)`**: Called before the test and its fixtures are set up.
-   **`pytest_runtest_call(item)`**: Called to execute the test function itself.
-   **`pytest_runtest_teardown(item, nextitem)`**: Called after the test has run and fixtures have been torn down.
-   **`pytest_runtest_makereport(item, call)`**: A crucial hook called to create the test report object. The `call` object contains information about whether the setup, call, or teardown phase failed. You can inspect the outcome (passed, failed, skipped) and add extra information to the report, which can then be used by other hooks or plugins (like `pytest-html`).

#### 4. Reporting Hooks
These hooks are used to customize the output of the test run.

-   **`pytest_report_header(config)`**: Allows you to add extra lines to the header of the test report.
-   **`pytest_terminal_summary(terminalreporter, exitstatus, config)`**: Called just before pytest exits. This is an alternative to `pytest_sessionfinish` that is specifically for adding content to the terminal report. We could have used this for our timer plugin.

By combining these hooks, you can create incredibly sophisticated plugins that integrate deeply with pytest's lifecycle. The official pytest documentation contains a complete reference of all available hooks. When building a plugin, your first step should be to browse this list to find the correct entry points for the logic you want to implement.

## Common Plugin Use Cases

## What Can You Build with Plugins?

Now that we understand the mechanics of hooks, let's explore some common and powerful use cases for custom plugins.

### Adding Custom Command-Line Options

Imagine our data validator needs to run against different environments (e.g., `staging`, `production`), and the data source changes for each. We can add a custom `--env` option to control this.

**`tests/conftest.py`**

```python
# tests/conftest.py

def pytest_addoption(parser):
    """Adds a custom --env command-line option."""
    parser.addoption(
        "--env", action="store", default="staging", help="environment to run tests against"
    )

@pytest.fixture
def api_endpoint(request):
    """A fixture that provides the correct API endpoint based on --env."""
    env = request.config.getoption("--env")
    if env == "staging":
        return "https://staging-api.example.com"
    elif env == "production":
        return "https://api.example.com"
    return "https://dev-api.example.com"

# In a test:
# def test_api_connection(api_endpoint):
#     assert requests.get(api_endpoint).status_code == 200
```

Now you can run `pytest --env=production` to target the production API. This pattern is fundamental for building flexible test suites.

### Dynamically Selecting or Reordering Tests

The `pytest_collection_modifyitems` hook gives you ultimate control over which tests run and in what order.

**Use Case**: You want to run all tests marked as `slow` at the very end of the test suite to get faster feedback on quick unit tests.

**`tests/conftest.py`**

```python
# tests/conftest.py

def pytest_collection_modifyitems(config, items):
    """Reorders tests to run @pytest.mark.slow tests last."""
    slow_tests = []
    other_tests = []
    for item in items:
        if item.get_closest_marker("slow"):
            slow_tests.append(item)
        else:
            other_tests.append(item)
    
    # Put the other tests first, then the slow ones
    items[:] = other_tests + slow_tests
```

This simple plugin automatically prioritizes your faster tests, improving the developer feedback loop without any manual intervention.

### Integrating with External Systems

Plugins are the perfect way to integrate your test suite with external tools like test case management systems (e.g., TestRail, Zephyr) or notification services (e.g., Slack).

**Use Case**: Post a summary of test failures to a Slack channel after a CI run.

**`tests/conftest.py` (conceptual)**

```python
# tests/conftest.py
import os
import requests

def pytest_sessionfinish(session):
    """Posts a summary to Slack on failure."""
    if session.testsfailed > 0 and os.getenv("CI"):
        slack_webhook_url = os.getenv("SLACK_WEBHOOK_URL")
        if not slack_webhook_url:
            return

        summary = f"Test run finished with {session.testsfailed} failures."
        # In a real implementation, you would list the failed tests.
        
        payload = {"text": summary}
        try:
            requests.post(slack_webhook_url, json=payload)
        except requests.RequestException as e:
            print(f"Failed to send Slack notification: {e}")
```

This hook runs at the end of the session, checks for failures, and (if running in a CI environment) sends a notification. This kind of automation is a hallmark of a mature testing setup.

## Distributing Your Own Plugin

## From `conftest.py` to Installable Package

Our custom timer plugin is useful, and we might want to use it in other projects. Copying the `conftest.py` file everywhere is not a scalable or maintainable solution. The professional approach is to package it as a proper, installable Python package.

### Iteration 5: Packaging the Custom Timer Plugin

**Current Limitation**: The plugin code lives inside our project's `tests/conftest.py`, making it impossible to share with other projects without copy-pasting.

**Goal**: Create a new, separate Python package for our `pytest-timer` plugin that can be installed with `pip`.

#### Step 1: Create the Package Structure

First, we'll create a new directory for our plugin project.

```
pytest-timer/
├── pyproject.toml
└── src/
    └── pytest_timer/
        ├── __init__.py
        └── plugin.py
```

-   `pyproject.toml`: The modern standard for defining Python package metadata and dependencies.
-   `src/pytest_timer/plugin.py`: We will move our plugin code here from `conftest.py`.

#### Step 2: Move the Plugin Code

Copy the hook implementations from `conftest.py` into `src/pytest_timer/plugin.py`. The code remains identical.

**`src/pytest_timer/plugin.py`**

```python
import time

def pytest_runtest_setup(item):
    """Hook called before a test item is run."""
    item.start_time = time.time()

def pytest_runtest_teardown(item, nextitem):
    """Hook called after a test item is run."""
    item.end_time = time.time()
    item.duration = item.end_time - item.start_time

def pytest_sessionfinish(session):
    """Hook called after the entire test session finishes."""
    reporter = session.config.pluginmanager.get_plugin('terminalreporter')
    reporter.write_sep("=", "CUSTOM TEST DURATIONS REPORT")
    
    all_items = [item for item in session.items if hasattr(item, 'duration')]
    slowest_tests = sorted(all_items, key=lambda x: x.duration, reverse=True)
    
    reporter.write_line("\nTop 5 slowest tests:")
    for item in slowest_tests[:5]:
        reporter.write_line(f"{item.nodeid}: {item.duration:.4f}s")
```

#### Step 3: Define the Package Metadata and Entry Point

This is the most critical step. We need to tell the Python packaging tools about our project and, most importantly, tell pytest that this package contains a plugin. This is done using an **entry point**.

The entry point is a special piece of metadata that tells a host application (like pytest) where to find plugins. For pytest, the entry point group is called `pytest11`.

**`pyproject.toml`**

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "pytest-timer"
version = "0.1.0"
authors = [
  { name="Your Name", email="you@example.com" },
]
description = "A simple pytest plugin to report slow tests"
requires-python = ">=3.8"
classifiers = [
    "Programming Language :: Python :: 3",
    "License :: OSI Approved :: MIT License",
    "Operating System :: OS Independent",
    "Framework :: Pytest",
]
dependencies = [
    "pytest",
]

[project.entry-points."pytest11"]
timer = "pytest_timer.plugin"
```

Let's break down the `[project.entry-points."pytest11"]` section:
-   `"pytest11"`: This is the specific entry point group that pytest looks for. The name is historical.
-   `timer`: This is a name we give our plugin. It's not strictly required but is good practice.
-   `"pytest_timer.plugin"`: This is the crucial part. It tells pytest to load the module `pytest_timer.plugin` as a plugin. Pytest will then inspect this module for any functions named like `pytest_*` and register them as hooks.

#### Step 4: Install and Verify

Now, we can install our plugin in editable mode from the root of the `pytest-timer` directory. This links the package into our Python environment so we can use it immediately.

```bash
# From within the pytest-timer/ directory
pip install -e .
```

Go back to our original data validator project. **Crucially, remove the plugin code from `tests/conftest.py`** so we can be sure the packaged version is being used.

Now, run pytest.

```bash
# In the original project directory
$ pytest
# ... (normal test output) ...

――――――――――――――――――――――――――――――――――― summary ――――――――――――――――――――――――――――――――――――
✅ 4 passed in 1.55s

======================= CUSTOM TEST DURATIONS REPORT =======================

Top 5 slowest tests:
tests/test_validator.py::test_valid_data_passes_all_checks: 0.3812s
tests/test_validator.py::test_duplicate_id_fails: 0.1540s
tests/test_validator.py::test_value_out_of_range_fails: 0.1233s
tests/test_validator.py::test_incorrect_row_length_fails: 0.1025s
```

It works! Pytest discovered our installed `pytest-timer` package via the entry point, loaded `pytest_timer.plugin`, and registered our hooks. Our project's test suite is now clean, and our plugin is a reusable, distributable tool that can be shared, versioned, and even published to the Python Package Index (PyPI) for anyone to use.

### The Journey: From Problem to Solution

| Iteration | Failure Mode / Problem                | Technique Applied                               | Result                                         |
| --------- | ------------------------------------- | ----------------------------------------------- | ---------------------------------------------- |
| 0         | Default output is verbose and slow    | None                                            | Baseline performance and readability           |
| 1         | Output is hard to scan quickly        | `pytest-sugar` plugin                           | Improved terminal UI with progress bar         |
| 2         | No shareable report for stakeholders  | `pytest-html` plugin                            | Generated self-contained HTML report           |
| 3         | Test suite execution is slow          | `pytest-xdist` plugin                           | Parallel execution, significantly faster suite |
| 4         | Need custom per-test timing data      | Local plugin in `conftest.py` using hooks       | Custom summary report with slow test warnings  |
| 5         | Custom plugin is not reusable         | Packaging plugin as an installable distribution | Plugin can be installed and used in any project|

This journey encapsulates the power of the pytest ecosystem. You can start by leveraging the work of others to quickly solve common problems. As your needs become more specific, you can tap into the same hook system the community uses to build your own powerful, project-specific tooling. And finally, you can contribute back to that ecosystem by packaging and sharing your solutions.
