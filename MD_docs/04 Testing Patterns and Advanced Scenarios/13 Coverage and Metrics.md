# Chapter 13: Coverage and Metrics

## Introduction to Code Coverage

## What is Code Coverage?

Imagine you've written a detailed set of instructions for a robot to navigate a maze. You send the robot in, and it successfully finds the exit. Your instructions work! But a question remains: did the robot follow every single path you laid out, or did it only use the main corridors, leaving entire sections of the maze unexplored?

**Code coverage** is the tool that answers this question for your software. It measures which lines of your application code are executed by your test suite. It doesn't tell you if your tests are *good*, but it tells you what parts of your code your tests *didn't even touch*.

### Why Does It Matter?

Coverage is a powerful diagnostic tool for three main reasons:

1.  **It reveals untested code:** The most immediate benefit is seeing exactly which functions, branches, or statements are not exercised by any of your tests. These are blind spots in your quality assurance.
2.  **It provides a safety net against dead code:** If a block of code has 0% coverage and has been that way for a long time, it might be "dead code"—legacy logic that is no longer used and can potentially be removed, simplifying your codebase.
3.  **It guides your testing efforts:** A coverage report acts like a map, highlighting areas that need more attention. Instead of guessing where to write the next test, you can focus on the parts of your application with the lowest coverage.

### The Critical Limitation: Coverage is Not a Goal

It's tempting to see a metric like "95% coverage" and treat it as a grade. This is a dangerous trap. A test can execute a line of code without actually verifying its behavior.

Consider this function:

```python
# src/calculator.py
def add(a, b):
    # A subtle bug: this should be a + b
    print(f"Adding {a} and {b}")
    return a - b
```

And this test:

```python
# tests/test_calculator.py
from src.calculator import add

def test_add_runs():
    add(5, 10) # We call the function, but don't check the result!
```

This test will give you **100% coverage** for the `add` function because every line was executed. However, the test is useless—it would pass even though the function is completely broken.

Remember the core principle: **Coverage tells you what you *haven't* tested; it doesn't tell you how *well* you've tested it.**

Throughout this chapter, we'll learn how to use coverage as a tool to guide our testing, not as a target to be blindly pursued.

## Installing and Using pytest-cov

## Installing and Using pytest-cov

The most popular tool for measuring code coverage with pytest is a plugin called `pytest-cov`. It integrates seamlessly with pytest and the underlying `coverage.py` library.

### Installation

Installation is a single command using pip. Make sure your virtual environment is activated first.

```bash
pip install pytest-cov
```

That's it. The plugin is now available to pytest.

### A Simple Project to Test

Let's create a small project to see `pytest-cov` in action. Our project will validate user account information.

**Project Structure:**
```
pytest_project/
├── src/
│   └── user_validator.py
└── tests/
    └── test_user_validator.py
```

Here is the application code:

```python
# src/user_validator.py

def is_valid_username(username: str) -> bool:
    """
    Checks if a username is valid.
    - Must be between 3 and 20 characters.
    - Must contain only alphanumeric characters.
    """
    if not 3 <= len(username) <= 20:
        return False
    if not username.isalnum():
        return False
    return True

def is_strong_password(password: str) -> bool:
    """
    Checks if a password is strong.
    - Must be at least 8 characters long.
    - Must contain at least one digit.
    """
    if len(password) < 8:
        return False
    if not any(char.isdigit() for char in password):
        return False
    return True
```

And here is our initial test file, which only tests the username validation:

```python
# tests/test_user_validator.py
from src.user_validator import is_valid_username

def test_is_valid_username_happy_path():
    assert is_valid_username("testuser123") is True

def test_is_valid_username_too_short():
    assert is_valid_username("hi") is False

def test_is_valid_username_contains_symbols():
    assert is_valid_username("user-name!") is False
```

### Running Your First Coverage Report

To run pytest with coverage, you use the `--cov` flag. You should specify the package or directory containing your source code. In our case, that's `src`.

```bash
pytest --cov=src
```

When you run this, you'll see the standard pytest output, followed by a new coverage summary table:
```
============================= test session starts ==============================
...
tests/test_user_validator.py ...                                         [100%]

----------- coverage: platform linux, python 3.10.4-final-0 -----------
Name                      Stmts   Miss  Cover
---------------------------------------------
src/user_validator.py        12      4    67%
---------------------------------------------
TOTAL                        12      4    67%

============================== 3 passed in 0.01s ===============================
```
Instantly, we have valuable data. Our test suite executed 12 statements in `user_validator.py`, but missed 4 of them, resulting in 67% coverage. We can see at a glance that our `is_strong_password` function is completely untested.

## Understanding Coverage Reports

## Understanding Coverage Reports

The default terminal report is great for a quick summary, but to really dig into the details, `pytest-cov` can generate much richer reports.

### The Terminal Report Explained

Let's look at that table again:

```
Name                      Stmts   Miss  Cover
---------------------------------------------
src/user_validator.py        12      4    67%
```

-   **Name:** The file being analyzed.
-   **Stmts:** The total number of executable statements in the file. Comments and blank lines are not counted.
-   **Miss:** The number of statements that were *not* executed by any test.
-   **Cover:** The coverage percentage, calculated as `(Stmts - Miss) / Stmts`.

You can add a `--cov-report term-missing` flag to also see which line numbers were missed, right in your terminal.

```bash
pytest --cov=src --cov-report term-missing
```

The output will now include a `Missing` column:
```
----------- coverage: platform linux, python 3.10.4-final-0 -----------
Name                      Stmts   Miss  Cover   Missing
-------------------------------------------------------
src/user_validator.py        12      4    67%   16-22
-------------------------------------------------------
TOTAL                        12      4    67%

============================== 3 passed in 0.01s ===============================
```
The `Missing` column tells us that lines 16 through 22 were not executed. This corresponds exactly to our `is_strong_password` function.

### The Interactive HTML Report

The most powerful tool for analyzing coverage is the HTML report. It generates an interactive website that visualizes exactly which lines were covered and which were missed in your source code.

Generate it with the `--cov-report=html` flag.

```bash
pytest --cov=src --cov-report=html
```

This command creates a new directory named `htmlcov/` in your project root. Open the `index.html` file inside it with your web browser.

```bash
# On macOS
open htmlcov/index.html

# On Linux
xdg-open htmlcov/index.html

# On Windows
start htmlcov/index.html
```

You will see a summary page. Clicking on `src/user_validator.py` takes you to a detailed view of the source file.

-   **Green lines** were executed by your tests.
-   **Red lines** were missed.
-   **Gray lines** (like comments or docstrings) are not executable and are ignored.

This visual feedback is incredibly intuitive. You can immediately see the entire `is_strong_password` function is red, confirming it's completely untested. This report is the primary tool you'll use to identify and analyze coverage gaps.

## Coverage as a Quality Gate

## Coverage as a Quality Gate

One of the most powerful applications of coverage metrics is to enforce a minimum testing standard in your project, especially in an automated environment like a CI/CD pipeline (see Chapter 16). You can configure pytest to fail the entire test suite if coverage drops below a certain threshold. This is called a "quality gate."

### Failing the Build on Low Coverage

The `--cov-fail-under` flag tells `pytest-cov` to exit with a non-zero status code (which signals failure to automation tools) if the total coverage is less than the specified minimum.

Let's try to enforce 80% coverage on our project. We know our current coverage is 67%, so this command should fail.

```bash
pytest --cov=src --cov-fail-under=80
```

The test run will proceed as usual, but at the end, you'll see a new error message:
```
...
----------- coverage: platform linux, python 3.10.4-final-0 -----------
Name                      Stmts   Miss  Cover
---------------------------------------------
src/user_validator.py        12      4    67%
---------------------------------------------
TOTAL                        12      4    67%
FAIL: Coverage less than configured fail-under=80% (is 67%)
=========================== 1 failed, 3 passed in 0.02s ============================
```
Pytest reports a failure, even though all our tests passed! This is the quality gate in action. It prevents you from merging code that reduces the overall test coverage of the project.

### Meeting the Threshold

Let's add tests for `is_strong_password` to meet our 80% goal.

Add these tests to `tests/test_user_validator.py`:

```python
# tests/test_user_validator.py
# ... (existing tests) ...
from src.user_validator import is_strong_password

def test_is_strong_password_happy_path():
    assert is_strong_password("Str0ngP@ss!") is True

def test_is_strong_password_too_short():
    assert is_strong_password("abc") is False

def test_is_strong_password_no_digit():
    assert is_strong_password("StrongPassword") is False
```

Now, run the command again:

```bash
pytest --cov=src --cov-fail-under=80
```

The result is a resounding success:
```
============================= test session starts ==============================
...
tests/test_user_validator.py ......                                      [100%]

----------- coverage: platform linux, python 3.10.4-final-0 -----------
Name                      Stmts   Miss  Cover
---------------------------------------------
src/user_validator.py        12      0   100%
---------------------------------------------
TOTAL                        12      0   100%
--cov-fail-under is set to 80%, measured 100% - PASSED
============================== 6 passed in 0.01s ===============================
```
Our coverage is now 100%, easily clearing the 80% bar. The build passes. By setting this gate, you ensure that as the codebase grows, the test suite grows with it.

### Choosing a Threshold

Don't immediately set `--cov-fail-under=100`. Start with a realistic number based on your project's current state (e.g., 70% or 80%). The goal is to prevent coverage from *decreasing*. You can gradually increase the threshold over time as you improve your test suite.

## Coverage Gaps and Dead Code

## Coverage Gaps and Dead Code

Coverage reports are most useful for finding two things: code paths you forgot to test, and code that might not be used at all.

### Analyzing Branch Coverage Gaps

Let's modify our `is_valid_username` function to include a special case: admin users can have symbols.

```python
# src/user_validator.py

def is_valid_username(username: str) -> bool:
    """
    Checks if a username is valid.
    - Must be between 3 and 20 characters.
    - Must contain only alphanumeric characters, unless it's an admin.
    """
    if username.startswith("admin_"):
        return 3 <= len(username) <= 20  # Admins can have symbols

    if not 3 <= len(username) <= 20:
        return False
    if not username.isalnum():
        return False
    return True

# ... is_strong_password remains the same ...
```

Our existing tests don't know about this new "admin" logic. Let's run the coverage report again.

```bash
pytest --cov=src --cov-report=html
```

Now, when you view the HTML report for `user_validator.py`, you'll see something new. The line `if username.startswith("admin_"):` will be green, but the line `return 3 <= len(username) <= 20` will be red.

This is a **branch coverage gap**. Our tests executed the `if` condition, but it always evaluated to `False`, so the code inside the `if` block was never run. The report makes this gap obvious.

To fix it, we add a test for the admin case:

```python
# tests/test_user_validator.py
# ...

def test_is_valid_username_admin_can_have_symbols():
    assert is_valid_username("admin_user-1") is True
```

Running the coverage report again will show this line turning green, closing the gap.

### Identifying Dead Code

Sometimes, a coverage report reveals code that isn't just untested, but is actually unreachable. This might be a sign of "dead code" that can be safely removed. If you find a function with 0% coverage that no part of your application seems to call, it's a strong candidate for deletion. This helps keep your codebase clean and maintainable.

### Excluding Code from Coverage

Not all code needs to be tested. You might have debugging helpers, compatibility fallbacks for old Python versions, or abstract methods that are meant to be implemented by subclasses. Forcing 100% coverage in these cases is counterproductive.

You can tell `coverage.py` to ignore a line by adding a `# pragma: no cover` comment.

```python
def complex_debug_helper():
    # This function is only for interactive debugging sessions,
    # it's hard to test and not part of the core logic.
    import pdb; pdb.set_trace() # pragma: no cover

def get_os_specific_path():
    if sys.platform == "win32":
        return "C:\\Users\\Default"
    else:
        # We only run tests on Linux, so this branch is never hit.
        return "/home/default" # pragma: no cover
```

Use `pragma: no cover` judiciously. It's a declaration that you are *intentionally* not testing a piece of code. It should be used for code that is genuinely untestable or not part of the production logic, not as a shortcut to silence a failing coverage report.

## Achieving Meaningful Coverage (Not Just High Percentages)

## Achieving Meaningful Coverage (Not Just High Percentages)

We end this chapter with the most important lesson: chasing a 100% coverage score is a fool's errand. This leads to a phenomenon called **"Coverage Theater"**—writing low-value tests just to make the numbers look good, without actually improving software quality.

### The Pitfall: A Useless Test with 100% Coverage

Let's revisit our `user_validator` module. Imagine a developer is told they *must* achieve 100% coverage. They could write this single, terrible test:

```python
# tests/a_bad_test.py
from src import user_validator

def test_to_get_100_percent_coverage():
    # Call functions with various inputs to hit all lines
    user_validator.is_valid_username("testuser")
    user_validator.is_valid_username("ad")
    user_validator.is_valid_username("admin_user")
    user_validator.is_valid_username("bad-user")
    
    user_validator.is_strong_password("GoodPass123")
    user_validator.is_strong_password("short")
    user_validator.is_strong_password("nodigits")

    # No assertions!
    assert True
```

If you run a coverage report on this test, it will proudly announce **100% coverage**. Every line in `user_validator.py` was executed. Yet, this test provides **zero value**. If we introduced a bug into any of the validation functions, this test would still pass. It ensures the code runs without crashing, but it doesn't verify that the code is *correct*.

This is the danger of treating coverage as a goal.

### The Philosophy: Coverage as a Guide

The healthy way to use code coverage is as a diagnostic tool, not a success metric. Your goal is not "100% coverage"; your goal is a "well-tested, reliable application." Coverage helps you get there.

Follow this workflow:

1.  **Write tests that verify behavior.** Focus on the requirements. What should the function do with valid input? What should it do with invalid input? What about edge cases (empty strings, zero, `None`)? Write clear, focused tests with strong assertions for each behavior.
2.  **Run the coverage report.** After writing your behavior-driven tests, run the coverage report.
3.  **Analyze the gaps.** Look at the red lines in the HTML report. For each missed line or branch, ask yourself: **"What user behavior or system state corresponds to this piece of code?"**
4.  **Write a new test for that behavior.** The answer to the question above tells you what your next test should be. You're not writing a test "to make a line green." You're writing a test "to verify the admin username logic," which has the *side effect* of making the line green.

Coverage doesn't tell you if your assertions are good. It doesn't tell you if you've tested enough edge cases. It only tells you which parts of your code have *zero* tests. It's the starting point for investigation, not the final grade. A high coverage score is often a *byproduct* of a good test suite, not the purpose of it.
