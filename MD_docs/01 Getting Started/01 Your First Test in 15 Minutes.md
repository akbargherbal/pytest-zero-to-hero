# Chapter 1: Your First Test in 15 Minutes

## Why We Test (The Quick Case)

## Why We Test (The Quick Case)

Before we write a single line of test code, let's understand what problem we're solving. Testing isn't about following best practices or checking boxes—it's about **confidence**.

### The Problem: Code That Works... Until It Doesn't

Imagine you've written a function to calculate the total price of items in a shopping cart, including tax:

```python
def calculate_total(items, tax_rate):
    subtotal = sum(item['price'] for item in items)
    tax = subtotal * tax_rate
    return subtotal + tax
```

You run it manually a few times. It works. You ship it to production.

Three weeks later, a customer reports they were charged incorrectly. You investigate and discover that when someone adds an item with a negative price (a discount), your function produces the wrong result. The tax calculation becomes negative, which doesn't make sense.

**The core issue**: You tested the function by running it manually with a few examples, but you didn't systematically verify all the scenarios it needs to handle. When you fixed a bug last week, you didn't re-verify that the original scenarios still worked.

### What Automated Testing Gives You

Automated tests are **executable specifications** that verify your code behaves correctly. Once written, they:

1. **Run instantly**: No manual clicking through your application
2. **Run repeatedly**: Every time you change code, verify nothing broke
3. **Document behavior**: Tests show exactly what your code is supposed to do
4. **Enable refactoring**: Change implementation with confidence
5. **Catch regressions**: Prevent old bugs from reappearing

### The Testing Mindset

Testing is not about proving your code is perfect. It's about:

- **Specifying expected behavior**: "When I call this function with these inputs, I expect this output"
- **Documenting edge cases**: "What happens with empty input? Negative numbers? None?"
- **Creating a safety net**: "I can change this code and immediately know if I broke something"

In the next 15 minutes, you'll write your first automated test. By the end of this chapter, you'll have a working test that runs automatically and tells you whether your code behaves as expected.

## Installing Pytest

## Installing Pytest

Pytest is a Python testing framework that makes writing tests simple and powerful. Let's get it installed.

### Prerequisites

You need Python 3.7 or later. Check your version:

```bash
python --version
```

If you see `Python 3.7.x` or higher, you're ready. If not, install Python from [python.org](https://www.python.org/downloads/).

### Installation with pip

Pytest is installed using Python's package manager, pip:

```bash
pip install pytest
```

This installs pytest and its dependencies. The installation takes about 10-30 seconds depending on your internet connection.

### Verify Installation

Confirm pytest is installed correctly:

```bash
pytest --version
```

You should see output like:

```
pytest 7.4.3
```

The exact version number may differ—any version 7.x or later is fine for this book.

### Using a Virtual Environment (Recommended)

While not strictly required for this chapter, using a virtual environment is a professional practice that isolates your project's dependencies. Here's the quick setup:

```bash
# Create a virtual environment
python -m venv venv

# Activate it (macOS/Linux)
source venv/bin/activate

# Activate it (Windows)
venv\Scripts\activate

# Install pytest in the virtual environment
pip install pytest
```

You'll see `(venv)` appear in your terminal prompt, indicating the virtual environment is active. All subsequent `pip install` commands will install packages only in this isolated environment.

**Why this matters**: Different projects may need different versions of pytest or other libraries. Virtual environments prevent conflicts between projects.

For now, whether you use a virtual environment or install pytest globally, you're ready to write your first test.

## Writing Your First Test Function

## Writing Your First Test Function

Let's write a test for a real function. We'll start with something concrete: a function that validates email addresses.

### Phase 1: The Reference Implementation

Create a new file called `email_validator.py`:

```python
def is_valid_email(email):
    """Check if an email address is valid."""
    if '@' not in email:
        return False
    if email.count('@') != 1:
        return False
    username, domain = email.split('@')
    if not username or not domain:
        return False
    if '.' not in domain:
        return False
    return True
```

This function checks basic email validity: it must have exactly one `@` symbol, non-empty username and domain parts, and the domain must contain a dot.

Now let's write a test. Create a new file called `test_email_validator.py`:

```python
from email_validator import is_valid_email

def test_valid_email_returns_true():
    result = is_valid_email('user@example.com')
    assert result == True
```

Let's break down what makes this a test:

### Anatomy of a Test Function

**1. The filename starts with `test_`**

Pytest discovers tests by looking for files that match the pattern `test_*.py` or `*_test.py`. Our file `test_email_validator.py` follows this convention.

**2. The function name starts with `test_`**

Pytest looks for functions whose names start with `test_`. Our function `test_valid_email_returns_true()` will be automatically discovered and executed.

**3. The function takes no parameters**

Test functions are called by pytest without arguments. They're self-contained specifications of behavior.

**4. The function uses `assert`**

The `assert` statement is Python's built-in way to verify a condition is true. If the condition is false, Python raises an `AssertionError`. Pytest captures these errors and reports them as test failures.

### What This Test Verifies

This test specifies: "When I call `is_valid_email()` with a valid email address like `'user@example.com'`, the function should return `True`."

The test has three parts:

1. **Arrange**: Set up the input (`'user@example.com'`)
2. **Act**: Call the function and capture the result
3. **Assert**: Verify the result matches expectations

This pattern—Arrange, Act, Assert—is fundamental to testing. You'll see it in every test you write.

### Why We Import the Function

Notice the first line:

```python
from email_validator import is_valid_email
```

Tests import the code they're testing just like any other Python code. The test file and the code file are separate—this separation is intentional. Tests should be independent modules that verify your code's behavior from the outside.

## Running Your First Test

## Running Your First Test

Now comes the moment of truth: running the test to see if our code behaves as expected.

### Running Pytest

In your terminal, navigate to the directory containing both `email_validator.py` and `test_email_validator.py`, then run:

```bash
pytest
```

Pytest will automatically discover and run your test. You should see output like this:

```
============================= test session starts ==============================
platform darwin -- Python 3.11.5, pytest-7.4.3, pluggy-1.3.0
rootdir: /Users/you/project
collected 1 item

test_email_validator.py .                                                [100%]

============================== 1 passed in 0.01s ===============================
```

### Reading the Output

Let's parse this output section by section:

**1. Session header**:
```
============================= test session starts ==============================
platform darwin -- Python 3.11.5, pytest-7.4.3, pluggy-1.3.0
```

This tells you:
- What platform you're on (`darwin` = macOS, `linux`, or `win32`)
- Your Python version (3.11.5)
- Your pytest version (7.4.3)
- The plugin system version (pluggy-1.3.0)

**2. Collection summary**:
```
collected 1 item
```

Pytest found 1 test function to run. This confirms pytest successfully discovered your test.

**3. Test execution**:
```
test_email_validator.py .                                                [100%]
```

This line shows:
- The filename being tested (`test_email_validator.py`)
- A dot (`.`) representing one passing test
- The progress percentage (`[100%]`)

Each test that passes gets a dot. If a test fails, you'll see an `F` instead.

**4. Summary**:
```
============================== 1 passed in 0.01s ===============================
```

The final tally: 1 test passed, and it took 0.01 seconds to run.

### What Just Happened?

When you ran `pytest`, here's what occurred:

1. **Discovery**: Pytest scanned your directory for files matching `test_*.py`
2. **Collection**: It found `test_email_validator.py` and looked for functions starting with `test_`
3. **Execution**: It called `test_valid_email_returns_true()`
4. **Verification**: The `assert` statement passed (the result was indeed `True`)
5. **Reporting**: Pytest reported the test as passed

### Running Tests Explicitly

You can also run a specific test file:

```bash
pytest test_email_validator.py
```

Or even a specific test function:

```bash
pytest test_email_validator.py::test_valid_email_returns_true
```

The `::` syntax tells pytest to run only that specific function. This is useful when you have many tests and want to focus on one.

### Verbose Output

For more detailed information, add the `-v` flag:

```bash
pytest -v
```

This produces:

```
============================= test session starts ==============================
platform darwin -- Python 3.11.5, pytest-7.4.3, pluggy-1.3.0
rootdir: /Users/you/project
collected 1 item

test_email_validator.py::test_valid_email_returns_true PASSED           [100%]

============================== 1 passed in 0.01s ===============================
```

Notice the difference: instead of a dot, you see the full test name and the word `PASSED`. Verbose mode is helpful when you have many tests and want to see exactly which ones ran.

Congratulations! You've just run your first automated test. The test passed, which means your `is_valid_email()` function correctly handles the case of a valid email address.

## Understanding Test Results

## Understanding Test Results

A passing test is satisfying, but the real learning happens when tests fail. Let's intentionally break our code to see what pytest tells us.

### Iteration 1: Introducing a Bug

Let's modify our `is_valid_email()` function to introduce a bug. Change the function in `email_validator.py`:

```python
def is_valid_email(email):
    """Check if an email address is valid."""
    if '@' not in email:
        return False
    if email.count('@') != 1:
        return False
    username, domain = email.split('@')
    if not username or not domain:
        return False
    # Bug introduced: checking for '@' instead of '.'
    if '@' not in domain:
        return False
    return True
```

We changed the last validation from checking for a dot (`.`) in the domain to checking for an at-sign (`@`). This is clearly wrong—domains should contain dots, not at-signs.

Now run the test again:

```bash
pytest
```

You'll see:

```
============================= test session starts ==============================
platform darwin -- Python 3.11.5, pytest-7.4.3, pluggy-1.3.0
rootdir: /Users/you/project
collected 1 item

test_email_validator.py F                                                [100%]

=================================== FAILURES ===================================
_______________________ test_valid_email_returns_true __________________________

    def test_valid_email_returns_true():
        result = is_valid_email('user@example.com')
>       assert result == True
E       assert False == True

test_email_validator.py:5: AssertionError
=========================== short test summary info ============================
FAILED test_email_validator.py::test_valid_email_returns_true - assert False...
============================== 1 failed in 0.03s ===============================
```

### Diagnostic Analysis: Reading the Failure

Let's parse this failure message systematically. This is a critical skill—reading test failures is how you diagnose problems.

**1. The test execution line**:
```
test_email_validator.py F                                                [100%]
```

The `F` indicates a failed test. If you had multiple tests, you'd see a pattern like `.F..F.` showing which tests passed (`.`) and which failed (`F`).

**2. The FAILURES section header**:
```
=================================== FAILURES ===================================
_______________________ test_valid_email_returns_true __________________________
```

This section contains detailed information about each failure. The underscored line shows which test failed.

**3. The code context**:
```
    def test_valid_email_returns_true():
        result = is_valid_email('user@example.com')
>       assert result == True
E       assert False == True
```

Pytest shows you:
- The test function that failed
- The line where the failure occurred (marked with `>`)
- What the assertion was checking (`assert result == True`)
- What actually happened (`assert False == True`)

The `E` line is pytest's **assertion introspection**. Pytest doesn't just tell you the assertion failed—it shows you the actual values involved. Here, `result` was `False`, but we expected `True`.

**4. The location**:
```
test_email_validator.py:5: AssertionError
```

This tells you the exact file and line number where the assertion failed. Line 5 in `test_email_validator.py` is where the problem was detected.

**5. The summary**:
```
=========================== short test summary info ============================
FAILED test_email_validator.py::test_valid_email_returns_true - assert False...
============================== 1 failed in 0.03s ===============================
```

A concise summary of what failed. This is especially useful when you have many tests—you can quickly scan to see which ones need attention.

### What This Tells Us

The failure message reveals:

1. **What we expected**: `result == True`
2. **What we got**: `result` was `False`
3. **Where it failed**: Line 5 of `test_email_validator.py`

This is enough information to start debugging. We know the function returned `False` when we expected `True`. Now we need to figure out why.

### Fixing the Bug

Let's fix the bug in `email_validator.py`:

```python
def is_valid_email(email):
    """Check if an email address is valid."""
    if '@' not in email:
        return False
    if email.count('@') != 1:
        return False
    username, domain = email.split('@')
    if not username or not domain:
        return False
    # Fixed: checking for '.' in domain
    if '.' not in domain:
        return False
    return True
```

Run the test again:

```bash
pytest
```

```
============================= test session starts ==============================
platform darwin -- Python 3.11.5, pytest-7.4.3, pluggy-1.3.0
rootdir: /Users/you/project
collected 1 item

test_email_validator.py .                                                [100%]

============================== 1 passed in 0.01s ===============================
```

The test passes again. The dot (`.`) confirms our fix worked.

### Iteration 2: Testing Invalid Input

Our test only verifies that valid emails return `True`. What about invalid emails? Let's add a test for that case.

Add this function to `test_email_validator.py`:

```python
def test_invalid_email_returns_false():
    result = is_valid_email('invalid-email')
    assert result == False
```

Run pytest:

```bash
pytest
```

```
============================= test session starts ==============================
platform darwin -- Python 3.11.5, pytest-7.4.3, pluggy-1.3.0
rootdir: /Users/you/project
collected 2 items

test_email_validator.py ..                                               [100%]

============================== 2 passed in 0.01s ===============================
```

Now we have two tests (two dots), and both pass. We've verified two scenarios:

1. Valid emails return `True`
2. Invalid emails (missing `@`) return `False`

### The Pattern of Test-Driven Confidence

Notice the workflow that's emerging:

1. **Write a test** that specifies expected behavior
2. **Run the test** to see if the code behaves correctly
3. **If it fails**, read the failure message to understand what went wrong
4. **Fix the code** to make the test pass
5. **Run the test again** to verify the fix worked

This cycle—write test, run test, fix code, verify—is the foundation of test-driven development. Each test you write increases your confidence that the code works as intended.

### Understanding Assertion Introspection

Pytest's assertion introspection is one of its most powerful features. Let's see it in action with a more complex example.

Add this test to `test_email_validator.py`:

```python
def test_email_with_multiple_at_signs_returns_false():
    result = is_valid_email('user@@example.com')
    assert result == False
```

This test verifies that emails with multiple `@` symbols are rejected. Run it:

```bash
pytest -v
```

```
============================= test session starts ==============================
platform darwin -- Python 3.11.5, pytest-7.4.3, pluggy-1.3.0
rootdir: /Users/you/project
collected 3 items

test_email_validator.py::test_valid_email_returns_true PASSED           [ 33%]
test_email_validator.py::test_invalid_email_returns_false PASSED        [ 66%]
test_email_validator.py::test_email_with_multiple_at_signs_returns_false PASSED [100%]

============================== 3 passed in 0.01s ===============================
```

All three tests pass. The verbose output (`-v`) shows each test name and its result.

### When Assertions Get Complex

Let's see what happens with a more detailed assertion. Add this test:

```python
def test_email_parts_are_extracted_correctly():
    email = 'john.doe@company.com'
    username, domain = email.split('@')
    assert username == 'john.doe'
    assert domain == 'company.com'
```

This test verifies that splitting an email on `@` produces the expected parts. Now let's intentionally break it to see pytest's introspection. Change the test to:

```python
def test_email_parts_are_extracted_correctly():
    email = 'john.doe@company.com'
    username, domain = email.split('@')
    assert username == 'jane.doe'  # Wrong expected value
    assert domain == 'company.com'
```

Run pytest:

```bash
pytest test_email_validator.py::test_email_parts_are_extracted_correctly -v
```

```
============================= test session starts ==============================
platform darwin -- Python 3.11.5, pytest-7.4.3, pluggy-1.3.0
rootdir: /Users/you/project
collected 1 item

test_email_validator.py::test_email_parts_are_extracted_correctly FAILED [100%]

=================================== FAILURES ===================================
__________________ test_email_parts_are_extracted_correctly ____________________

    def test_email_parts_are_extracted_correctly():
        email = 'john.doe@company.com'
        username, domain = email.split('@')
>       assert username == 'jane.doe'
E       AssertionError: assert 'john.doe' == 'jane.doe'
E         - jane.doe
E         + john.doe

test_email_validator.py:17: AssertionError
=========================== short test summary info ============================
FAILED test_email_validator.py::test_email_parts_are_extracted_correctly
============================== 1 failed in 0.03s ===============================
```

### Diagnostic Analysis: String Comparison

Look at pytest's introspection:

```
E       AssertionError: assert 'john.doe' == 'jane.doe'
E         - jane.doe
E         + john.doe
```

Pytest shows you:
1. The actual comparison that failed
2. A diff-style view showing what was expected (`-`) vs. what was actual (`+`)

This is incredibly helpful for debugging. You don't have to add print statements or use a debugger—pytest shows you exactly what values were involved in the failed assertion.

Fix the test by changing the expected value back to `'john.doe'`, and it will pass again.

### Key Takeaways

When a test fails, pytest gives you:

1. **The exact location** (file and line number)
2. **The assertion that failed** (the actual code)
3. **The values involved** (assertion introspection)
4. **A comparison** (for complex types like strings, lists, dicts)

This information is usually sufficient to diagnose the problem without additional debugging tools.

## What You've Accomplished

## What You've Accomplished

In less than 15 minutes, you've learned the fundamentals of automated testing with pytest. Let's review what you now know.

### Skills Acquired

**1. Installation and Setup**
- Installed pytest using pip
- Verified the installation
- Understood the role of virtual environments

**2. Writing Tests**
- Created a test file following pytest's naming conventions (`test_*.py`)
- Wrote test functions that start with `test_`
- Used Python's `assert` statement to verify behavior
- Applied the Arrange-Act-Assert pattern

**3. Running Tests**
- Executed tests with the `pytest` command
- Ran specific test files and functions
- Used verbose mode (`-v`) for detailed output

**4. Reading Test Results**
- Interpreted passing test output (dots and summary)
- Analyzed failing test output (failure details and assertion introspection)
- Used pytest's diagnostic information to identify bugs
- Fixed code based on test failures

### The Complete Test Suite

Here's your final `test_email_validator.py`:

```python
from email_validator import is_valid_email

def test_valid_email_returns_true():
    result = is_valid_email('user@example.com')
    assert result == True

def test_invalid_email_returns_false():
    result = is_valid_email('invalid-email')
    assert result == False

def test_email_with_multiple_at_signs_returns_false():
    result = is_valid_email('user@@example.com')
    assert result == False

def test_email_parts_are_extracted_correctly():
    email = 'john.doe@company.com'
    username, domain = email.split('@')
    assert username == 'john.doe'
    assert domain == 'company.com'
```

And your `email_validator.py`:

```python
def is_valid_email(email):
    """Check if an email address is valid."""
    if '@' not in email:
        return False
    if email.count('@') != 1:
        return False
    username, domain = email.split('@')
    if not username or not domain:
        return False
    if '.' not in domain:
        return False
    return True
```

### The Testing Mindset

You've internalized the core testing workflow:

1. **Specify behavior**: Write a test that describes what should happen
2. **Verify behavior**: Run the test to check if the code does what you expect
3. **Diagnose failures**: Read pytest's output to understand what went wrong
4. **Fix and verify**: Correct the code and re-run tests to confirm the fix

This workflow scales from simple functions to complex systems. The principles remain the same.

### What's Next

You now have a foundation in pytest, but there's much more to learn:

- **Chapter 2** will deepen your understanding of assertions and test discovery
- **Chapter 3** will show you how to organize tests in real projects
- **Chapter 4** will introduce fixtures for managing test setup and teardown
- **Chapter 5** will teach you how to test multiple scenarios efficiently with parametrization

But for now, you've accomplished something significant: you've written automated tests that verify your code works correctly. Every time you change `email_validator.py`, you can run `pytest` and immediately know if you broke anything.

That's the power of automated testing. That's why we test.

### Try It Yourself

Before moving to the next chapter, try these exercises:

1. **Add a test** for an email with no domain (e.g., `'user@'`)
2. **Add a test** for an email with no username (e.g., `'@example.com'`)
3. **Add a test** for an email with spaces (e.g., `'user @example.com'`)
4. **Modify `is_valid_email()`** to handle these cases correctly
5. **Run pytest** to verify all tests pass

These exercises will reinforce what you've learned and prepare you for the deeper concepts ahead.

Congratulations on writing your first tests. You're on your way from zero to hero.
