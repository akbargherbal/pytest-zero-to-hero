# Chapter 17: Debugging Failing Tests

## Reading Test Output

## Reading Test Output

A failing test is not a failure; it's a success. It has successfully found a bug. The output from a failing test is not an error message to be feared, but a detailed map leading you directly to the problem. Learning to read this map is the most critical debugging skill you can develop.

Let's start with a simple, broken function and a test that exposes the bug.

### The Scenario: A Buggy Function

Imagine a function that's supposed to format a user's full name but has a subtle bug: it adds an extra space before the last name.

```python
# src/user_profile.py

def format_full_name(first_name: str, last_name: str) -> str:
    """Joins first and last names into a full name."""
    # Intentional bug: an extra space is added before the last name.
    return f"{first_name}  {last_name}"
```

```python
# tests/test_user_profile.py

from src.user_profile import format_full_name

def test_format_full_name():
    """Tests the full name formatting."""
    first = "Ada"
    last = "Lovelace"
    
    expected = "Ada Lovelace"
    result = format_full_name(first, last)
    
    assert result == expected
```

When we run this test, it will fail. Let's run pytest and dissect the output piece by piece.

```bash
$ pytest
=========================== test session starts ============================
platform linux -- Python 3.10.6, pytest-7.1.2, pluggy-1.0.0
rootdir: /path/to/project
collected 1 item

tests/test_user_profile.py F                                         [100%]

================================= FAILURES =================================
___________________________ test_format_full_name ____________________________

    def test_format_full_name():
        """Tests the full name formatting."""
        first = "Ada"
        last = "Lovelace"
    
        expected = "Ada Lovelace"
        result = format_full_name(first, last)
    
>       assert result == expected
E       AssertionError: assert 'Ada  Lovelace' == 'Ada Lovelace'
E         - Ada Lovelace
E         ?    ^
E         + Ada  Lovelace
E         ?    ^

tests/test_user_profile.py:11: AssertionError
========================= short test summary info ==========================
FAILED tests/test_user_profile.py::test_format_full_name - AssertionError...
============================ 1 failed in 0.02s =============================
```

This output is dense with information. Let's break it down.

### Anatomy of a Failure Report

1.  **Header**:
    ```
    =========================== test session starts ============================
    platform linux -- Python 3.10.6, pytest-7.1.2, pluggy-1.0.0
    rootdir: /path/to/project
    collected 1 item
    ```
    This section gives you context: your environment, Python version, pytest version, and the project's root directory. It also tells you how many tests it found (`collected 1 item`).

2.  **Progress Bar**:
    ```
    tests/test_user_profile.py F                                         [100%]
    ```
    Each character represents a test result. `.` is a pass, `F` is a failure, `E` is an error (an unexpected exception), `s` is a skip, and `x` is an expected failure. Here, we see our single test failed.

3.  **Failures Section**:
    ```
    ================================= FAILURES =================================
    ___________________________ test_format_full_name ____________________________
    ```
    This is the start of the detailed report for each failing test.

4.  **Traceback and Source Code**:
    ```python
        def test_format_full_name():
            ...
            result = format_full_name(first, last)
    
    >       assert result == expected
    ```
    Pytest shows you the exact line in your test file where the failure occurred, marked with a `>`. It also provides a few lines of context. If the failure happened deep inside another function call, you would see the full call stack here.

5.  **Assertion Introspection (The Magic)**:
    ```
    E       AssertionError: assert 'Ada  Lovelace' == 'Ada Lovelace'
    E         - Ada Lovelace
    E         ?    ^
    E         + Ada  Lovelace
    E         ?    ^
    ```
    This is where pytest shines. Instead of just telling you `False is not True`, it inspects the values involved in the failed assertion.
    - The line starting with `E` (for Error/Explanation) shows the exact assertion that failed.
    - It then provides a "diff" of the two strings. The `-` line is the expected value, the `+` line is the actual value (`result`), and the `?` lines highlight the exact character where they differ. We can immediately see the extra space in the actual result.

6.  **Failure Location**:
    ```
    tests/test_user_profile.py:11: AssertionError
    ```
    This line explicitly states the file, line number, and exception type for the failure.

7.  **Short Test Summary**:
    ```
    ========================= short test summary info ==========================
    FAILED tests/test_user_profile.py::test_format_full_name - AssertionError...
    ============================ 1 failed in 0.02s =============================
    ```
    Finally, a summary tells you how many tests failed, passed, etc., and the total time taken.

Treating this output as a structured report rather than a monolithic error message transforms debugging from a guessing game into a methodical process of analysis.

## Using pytest's Verbose and Extra-Verbose Modes

## Using pytest's Verbose and Extra-Verbose Modes

The default pytest output is concise, which is great for large test suites where you just want a quick overview. However, when debugging, you often need more detail. Pytest provides verbosity flags to control the level of output.

Let's create a slightly larger test file to see the difference.

```python
# tests/test_calculations.py

def add(a, b):
    return a + b

def subtract(a, b):
    # Intentional bug
    return a + b

def test_add_positive_numbers():
    assert add(2, 3) == 5

def test_add_negative_numbers():
    assert add(-2, -3) == -5

def test_subtract_numbers():
    assert subtract(5, 3) == 2
```

### Default Output (`pytest`)

Running `pytest` with no flags gives us the compact progress bar.

```bash
$ pytest tests/test_calculations.py
=========================== test session starts ============================
...
collected 3 items

tests/test_calculations.py ..F                                        [100%]

================================= FAILURES =================================
__________________________ test_subtract_numbers ___________________________

    def test_subtract_numbers():
>       assert subtract(5, 3) == 2
E       assert 8 == 2
E        +  where 8 = subtract(5, 3)

tests/test_calculations.py:15: AssertionError
========================= short test summary info ==========================
FAILED tests/test_calculations.py::test_subtract_numbers - assert 8 == 2
======================= 1 failed, 2 passed in 0.02s ========================
```

Notice the `..F` in the progress bar. Two passed, one failed. This is efficient but doesn't tell you *which* tests passed.

### Verbose Mode (`-v`)

The `-v` or `--verbose` flag changes the output to list each test individually with its result.

```bash
$ pytest -v tests/test_calculations.py
=========================== test session starts ============================
...
collected 3 items

tests/test_calculations.py::test_add_positive_numbers PASSED           [ 33%]
tests/test_calculations.py::test_add_negative_numbers PASSED           [ 66%]
tests/test_calculations.py::test_subtract_numbers FAILED               [100%]

================================= FAILURES =================================
__________________________ test_subtract_numbers ___________________________
... (failure details are the same) ...
========================= short test summary info ==========================
FAILED tests/test_calculations.py::test_subtract_numbers - assert 8 == 2
======================= 1 failed, 2 passed in 0.02s ========================
```

This is much clearer. We see the full node ID (`file::function`) for every test and its status (`PASSED` or `FAILED`). This is my personal default for running tests during development.

### Extra-Verbose Mode (`-vv`)

The `-vv` flag adds even more detail, primarily by showing more information during the setup and teardown phases of tests. For simple assertion failures like this, the output is often identical to `-v`. Its real power becomes apparent when you have complex fixtures or are debugging issues with the test runner itself.

### The Report Flag (`-r`)

The `-r` flag is a powerful companion to `-v`. It controls which information is shown in the "short test summary info" section. It takes a character argument to specify what to show. A common and highly useful combination is `-ra`.

- `r`: report
- `a`: all (except passes)

Let's run it on our file.

```bash
$ pytest -ra tests/test_calculations.py
=========================== test session starts ============================
...
collected 3 items

tests/test_calculations.py ..F                                        [100%]

================================= FAILURES =================================
... (full failure report) ...
========================= short test summary info ==========================
FAILED tests/test_calculations.py::test_subtract_numbers - assert 8 == 2
======================= 1 failed, 2 passed in 0.02s ========================
```

In this case, `-ra` doesn't add much because the default is to show failures. But if we had skipped tests or xfailed tests, they would also appear in this summary, giving a complete picture of the test run without the verbosity of `-v`.

**Common `-r` options:**
- `f`: failed
- `E`: error
- `s`: skipped
- `x`: xfailed
- `p`: passed
- `P`: passed with output
- `a`: all (except `p` and `P`)
- `A`: all

For daily debugging, `pytest -v` is your best friend. For CI systems or generating reports, combining `-r` with other flags can give you the exact summary you need.

## The -x Flag (Stop on First Failure)

## The -x Flag (Stop on First Failure)

When you run a large test suite, one single failure in a critical setup function (like connecting to a test database) can cause a cascade of hundreds of subsequent failures. This creates a huge amount of noise in your test report, burying the original, root-cause failure.

Pytest provides a simple and powerful solution: the `-x` (or `--exitfirst`) flag. It tells pytest to stop the entire test session immediately after the first test fails.

### The Problem: A Cascade of Failures

Let's create a scenario where an early test failure makes later tests irrelevant.

```python
# tests/test_workflow.py

def test_step_1_data_preparation():
    """This step prepares data, but it fails."""
    print("\nRunning Step 1: Data Preparation")
    assert False, "Data source is unavailable"

def test_step_2_data_processing():
    """This step depends on the data from step 1."""
    print("\nRunning Step 2: Data Processing")
    # This test would do something with the prepared data
    assert True

def test_step_3_generate_report():
    """This step generates a report from processed data."""
    print("\nRunning Step 3: Generate Report")
    # This test would use the processed data
    assert True
```

If `test_step_1` fails, the other two tests are meaningless. Let's see what happens when we run pytest normally. We'll use `-v` to see the individual tests and `-s` to see our `print` statements.

```bash
$ pytest -v -s tests/test_workflow.py
=========================== test session starts ============================
...
collected 3 items

tests/test_workflow.py::test_step_1_data_preparation 
Running Step 1: Data Preparation
FAILED               [ 33%]
tests/test_workflow.py::test_step_2_data_processing 
Running Step 2: Data Processing
PASSED               [ 66%]
tests/test_workflow.py::test_step_3_generate_report 
Running Step 3: Generate Report
PASSED               [100%]

================================= FAILURES =================================
______________________ test_step_1_data_preparation ______________________

    def test_step_1_data_preparation():
        """This step prepares data, but it fails."""
        print("\nRunning Step 1: Data Preparation")
>       assert False, "Data source is unavailable"
E       AssertionError: Data source is unavailable
E       assert False

tests/test_workflow.py:5: AssertionError
========================= short test summary info ==========================
FAILED tests/test_workflow.py::test_step_1_data_preparation - Assertio...
======================= 1 failed, 2 passed in 0.03s ========================
```

Pytest ran all three tests, even though the failure in `test_step_1` implies the others are running on invalid assumptions. This is just a small example; imagine this with 300 tests.

### The Solution: `pytest -x`

Now, let's run the same command but add the `-x` flag.

```bash
$ pytest -v -s -x tests/test_workflow.py
=========================== test session starts ============================
...
collected 3 items

tests/test_workflow.py::test_step_1_data_preparation 
Running Step 1: Data Preparation
FAILED               [ 33%]

================================= FAILURES =================================
______________________ test_step_1_data_preparation ______________________

    def test_step_1_data_preparation():
        """This step prepares data, but it fails."""
        print("\nRunning Step 1: Data Preparation")
>       assert False, "Data source is unavailable"
E       AssertionError: Data source is unavailable
E       assert False

tests/test_workflow.py:5: AssertionError
=========================== short test summary info ============================
FAILED tests/test_workflow.py::test_step_1_data_preparation - Assertio...
!!!!!!!!!!!!!!!!!!!!!!!!!! stopping after 1 failures !!!!!!!!!!!!!!!!!!!!!!!!!!!
============================ 1 failed in 0.02s =============================
```

The result is much cleaner.
- Pytest ran `test_step_1_data_preparation`.
- It failed.
- Pytest immediately stopped the session, printing `stopping after 1 failures`.
- `test_step_2` and `test_step_3` were never executed.

This allows you to focus your attention entirely on the first point of failure, which is almost always the root cause. The `-x` flag is an indispensable tool for efficient debugging in large codebases.

## The --pdb Flag (Drop into Debugger)

## The --pdb Flag (Drop into Debugger)

Sometimes, a failure message isn't enough. You need to inspect the state of your program—the values of variables, the call stack—at the exact moment of failure. This is called post-mortem debugging. Pytest integrates seamlessly with Python's built-in debugger, `pdb`, via the `--pdb` flag.

When you run tests with `pytest --pdb`, if a test fails or raises an uncaught exception, pytest will automatically drop you into an interactive `pdb` session at the point of failure.

### A Scenario for Debugging

Let's consider a function that processes a dictionary of user data. The bug is subtle and depends on the input data.

```python
# src/data_processor.py

def process_user_data(data: dict) -> dict:
    """Processes user data, calculating an age-based score."""
    processed = data.copy()
    
    # Bug: This assumes 'age' is always present and is an integer.
    # It will fail if 'age' is missing or a string.
    if processed['age'] > 30:
        processed['score'] = 100
    else:
        processed['score'] = 50
        
    processed['status'] = 'processed'
    return processed
```

```python
# tests/test_data_processor.py

from src.data_processor import process_user_data

def test_process_user_with_string_age():
    user_data = {
        "name": "Charlie",
        "age": "35",  # Age is a string, not an integer!
        "city": "New York"
    }
    
    processed = process_user_data(user_data)
    assert processed['score'] == 100
```

Running this test will result in a `TypeError` because you can't compare a string (`"35"`) with an integer (`30`). Let's use `--pdb` to investigate.

```bash
$ pytest --pdb tests/test_data_processor.py
=========================== test session starts ============================
...
collected 1 item

tests/test_data_processor.py E                                        [100%]

================================== ERRORS ==================================
_ ERROR at setup of test_process_user_with_string_age _

>   ???

/path/to/project/src/data_processor.py:6: in process_user_data
    if processed['age'] > 30:
E   TypeError: '>' not supported between instances of 'str' and 'int'

During handling of the above exception, another exception occurred:
...
tests/test_data_processor.py:12: TypeError
>>>>>>>>>>>>>>>>>>>>>>>>> entering PDB >>>>>>>>>>>>>>>>>>>>>>>>>
> /path/to/project/src/data_processor.py(6)process_user_data()
-> if processed['age'] > 30:
(Pdb)
```

Pytest has paused execution and dropped us into the `pdb` shell right at the line that caused the error. The `(Pdb)` prompt indicates we are in the debugger.

### Essential PDB Commands

Here are the most common commands you'll use:

| Command | Alias | Description                               |
|---------|-------|-------------------------------------------|
| `p <expr>` | `p`   | Print the value of an expression.         |
| `pp <expr>`| `pp`  | Pretty-print the value of an expression.  |
| `l`        | `l`   | List source code around the current line. |
| `n`        | `n`   | Execute the next line.                    |
| `c`        | `c`   | Continue execution until the next breakpoint or the end. |
| `q`        | `q`   | Quit the debugger and exit.               |
| `args`     | `a`   | Print the arguments of the current function. |
| `where`    | `w`   | Print the current call stack.             |

### An Interactive Debugging Session

Let's use these commands to figure out our bug.

1.  **Check the code context** with `l` (list):
    ```
    (Pdb) l
      1     def process_user_data(data: dict) -> dict:
      2         """Processes user data, calculating an age-based score."""
      3         processed = data.copy()
      4     
      5         # Bug: This assumes 'age' is always present and is an integer.
      6  ->     if processed['age'] > 30:
      7             processed['score'] = 100
      8         else:
      9             processed['score'] = 50
     10     
     11         processed['status'] = 'processed'
    ```
    The `->` arrow shows our exact location.

2.  **Inspect the variables** with `p` (print). Let's check the `processed` dictionary.
    ```
    (Pdb) p processed
    {'name': 'Charlie', 'age': '35', 'city': 'New York'}
    ```

3.  **Isolate the problem**. Let's inspect the specific value causing the `TypeError`.
    ```
    (Pdb) p processed['age']
    '35'
    (Pdb) p type(processed['age'])
    <class 'str'>
    ```
    Aha! The `TypeError` message told us we were comparing a `str` and an `int`, and now we've confirmed it. The value of `age` is the string `'35'`, not the integer `35`.

4.  **Quit the debugger** with `q` (quit).
    ```
    (Pdb) q
    ```

The `--pdb` flag gives you an interactive x-ray of your code at the moment of failure, making it one of the most powerful debugging tools in your arsenal.

## Using Breakpoints in Tests

## Using Breakpoints in Tests

The `--pdb` flag is fantastic for post-mortem debugging—analyzing the state *after* an error has occurred. But what if you want to pause execution *before* an error happens to inspect the state? For this, you can set a breakpoint directly in your code.

A breakpoint is a signal in your code that tells the debugger to pause execution at that specific line.

### Setting a Breakpoint

Since Python 3.7, the recommended way to set a breakpoint is with the built-in `breakpoint()` function. For older Python versions, you would use `import pdb; pdb.set_trace()`.

Let's modify our previous test to use a breakpoint. We'll place it right before the function call to inspect the data we're about to pass in.

```python
# tests/test_data_processor_breakpoint.py

from src.data_processor import process_user_data

def test_process_user_with_breakpoint():
    user_data = {
        "name": "Charlie",
        "age": "35",
        "city": "New York"
    }
    
    print("About to call process_user_data...")
    
    # Set a breakpoint here
    breakpoint()
    
    # The code will pause here before this next line is executed
    processed = process_user_data(user_data)
    assert processed['score'] == 100
```

Now, run pytest *without* the `--pdb` flag. Pytest's output capturing mechanism needs to be disabled for the interactive debugger to work correctly, so we use the `-s` flag.

```bash
$ pytest -s tests/test_data_processor_breakpoint.py
=========================== test session starts ============================
...
collected 1 item

tests/test_data_processor_breakpoint.py::test_process_user_with_breakpoint 
About to call process_user_data...
> /path/to/project/tests/test_data_processor_breakpoint.py(16)test_process_user_with_breakpoint()
-> processed = process_user_data(user_data)
(Pdb)
```

As you can see, the test execution paused exactly where we put `breakpoint()`, and we are now in a `pdb` session. We can inspect local variables just like before.

```bash
(Pdb) p user_data
{'name': 'Charlie', 'age': '35', 'city': 'New York'}
(Pdb) c
```

After inspecting, we type `c` (continue) to resume execution. The test will then continue, call `process_user_data`, and fail with the `TypeError` as before.

### `breakpoint()` vs. `pytest --pdb`

It's crucial to understand the difference between these two powerful tools:

-   **`pytest --pdb`**: **Reactive**. Automatically starts a debugger session *on failure*. You don't need to modify your code. It's for investigating *why* something failed.
-   **`breakpoint()`**: **Proactive**. You modify your code to explicitly tell the debugger *where* to pause. It's for inspecting the state at a specific point in your logic, regardless of whether an error has occurred yet.

### Using a Different Debugger

Pytest allows you to use alternative, more powerful debuggers like `ipdb` (from IPython) or `pudb` (a visual debugger). If you have `pytest-ipdb` installed, for example, you can use it by setting a configuration option or a command-line flag.

To use `ipdb` for the `--pdb` flag, you can run:

```bash
# First, install the plugin
pip install pytest-ipdb

# Run pytest with the custom debugger class
pytest --pdbcls=IPython.terminal.debugger:Pdb
```

Using `breakpoint()` will also respect the `PYTHONBREAKPOINT` environment variable, allowing you to switch debuggers without changing your code. For example:

```bash
export PYTHONBREAKPOINT=ipdb.set_trace
pytest -s tests/test_data_processor_breakpoint.py
```

This will launch `ipdb` instead of `pdb` when `breakpoint()` is called.

## Logging and Debugging Information

## Logging and Debugging Information

In complex applications, `print()` statements and debuggers aren't always enough. A robust logging setup is essential for understanding program flow. By default, pytest cleverly captures all output (including `stdout`, `stderr`, and log messages) to keep your test results clean. It only displays this captured output for failing tests.

### The Problem: Hidden Output

Let's write a function that logs its progress and a test for it.

```python
# src/reporter.py
import logging

log = logging.getLogger(__name__)

def generate_report(data: list) -> str:
    """Generates a summary report from a list of numbers."""
    log.info(f"Starting report generation for {len(data)} items.")
    
    if not data:
        log.warning("Input data is empty.")
        return "Empty Report"
        
    total = sum(data)
    log.debug(f"Calculated total: {total}")
    
    report = f"Report: {len(data)} items, Total = {total}"
    log.info("Report generation complete.")
    return report
```

```python
# tests/test_reporter.py
from src.reporter import generate_report

def test_generate_report_success():
    result = generate_report([10, 20, 30])
    assert "Total = 60" in result

def test_generate_report_empty():
    result = generate_report([])
    assert result == "Empty Report"
```

If you run this with `pytest`, both tests will pass, and you won't see any of the log messages. This is usually what you want. But for debugging, you need to see them.

### Viewing Log Messages

There are several ways to configure pytest to display logs during a test run.

1.  **Disable all capturing (`-s`)**: The simplest way is to use `pytest -s`. This disables all output capturing, so `print` statements and log messages will be printed to the console in real-time. This is a blunt instrument but effective for quick debugging.

2.  **Set the log level (`--log-cli-level`)**: A more controlled approach is to tell pytest the minimum log level to display on the console.

```bash
$ pytest --log-cli-level=INFO
=========================== test session starts ============================
...
collected 2 items

tests/test_reporter.py::test_generate_report_success 
------------------------------- live log call --------------------------------
INFO     src.reporter:reporter.py:7 Starting report generation for 3 items.
INFO     src.reporter:reporter.py:15 Report generation complete.
PASSED                                                                 [ 50%]
tests/test_reporter.py::test_generate_report_empty 
------------------------------- live log call --------------------------------
INFO     src.reporter:reporter.py:7 Starting report generation for 0 items.
WARNING  src.reporter:reporter.py:10 Input data is empty.
PASSED                                                                 [100%]

============================ 2 passed in 0.02s =============================
```

Notice the `DEBUG` message was not shown, because its level is lower than `INFO`. This is a much cleaner way to see logs without disabling all of pytest's helpful capturing.

### Testing Logs with the `caplog` Fixture

Viewing logs is useful for manual debugging, but what if you want to *test* that your code is logging correctly? For this, pytest provides the built-in `caplog` fixture.

The `caplog` fixture captures log records so you can make assertions against them.

Let's rewrite `test_generate_report_empty` to verify that a warning was logged.

```python
# tests/test_reporter.py (updated)
import logging
from src.reporter import generate_report

def test_generate_report_success(caplog):
    result = generate_report([10, 20, 30])
    assert "Total = 60" in result

def test_generate_report_empty_logs_warning(caplog):
    """Verify that a WARNING is logged for empty data."""
    # You can optionally set the level for the capture context
    with caplog.at_level(logging.WARNING):
        result = generate_report([])
    
    assert result == "Empty Report"
    
    # caplog.records is a list of all captured LogRecord objects
    assert len(caplog.records) == 1
    
    # Get the first record
    record = caplog.records[0]
    
    assert record.levelname == "WARNING"
    assert "Input data is empty" in record.message
```

Here, we've turned a side effect (logging) into a testable behavior. The `caplog` fixture gives you access to:
- `caplog.text`: All captured log messages as a single string.
- `caplog.records`: A list of `logging.LogRecord` objects.
- `caplog.record_tuples`: A list of `(logger_name, log_level, message)` tuples.

Using `caplog` is the idiomatic pytest way to interact with logs. It allows you to write precise tests that confirm your application's logging behavior, which is critical for observability and production support.
