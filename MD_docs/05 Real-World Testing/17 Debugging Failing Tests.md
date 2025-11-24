# Chapter 17: Debugging Failing Tests

## Reading Test Output

## The Debugging Mindset

A failing test is not a roadblock; it's a roadmap. Pytest provides a wealth of information when a test fails, and learning to read this output systematically is the most critical debugging skill you can develop. A test failure provides three key pieces of information: **what** failed, **where** it failed, and **why** it failed.

In this chapter, we will adopt the role of a detective. We'll start with a buggy piece of code and a failing test. With each new tool pytest offers, we will uncover more clues, moving from a simple crash report to a full interactive investigation.

### Phase 1: Establish the Reference Implementation

Our anchor example for this chapter will be a function designed to perform simple analytics on a list of user data. It's a common type of data-processing task, and we've introduced a few subtle bugs that are typical of real-world code.

This function, `analyze_user_ages`, is intended to:
1.  Calculate the average age of all users.
2.  Count how many users are adults (age 18 or over).
3.  Return a sorted list of user names formatted as "Last, First".

Here is the initial, buggy implementation.

```python
# user_analytics.py

def analyze_user_ages(users: list[dict]) -> dict:
    """
    Analyzes a list of user data, returning statistics.
    - Calculates average age.
    - Counts number of adults (age >= 18).
    """
    total_age = sum(user['age'] for user in users)
    num_users = len(users)
    average_age = total_age / num_users

    # Bug: Logic is incorrect for counting adults (should be >= 18)
    adult_count = 0
    for user in users:
        if user['age'] > 18:
            adult_count += 1

    # Bug: Assumes 'first' and 'last' keys will always exist
    user_names = [f"{user['last']}, {user['first']}" for user in users]

    return {
        "average_age": average_age,
        "adult_count": adult_count,
        "user_names": sorted(user_names)
    }
```

### Iteration 1: The Crash

Let's write our first test. We'll test the basic functionality with a valid list of users. However, to trigger our first bug, we'll include a user record that is missing the `first` name key. This is a common issue when dealing with inconsistent data sources.

```python
# test_analytics_v1.py
from user_analytics import analyze_user_ages

def test_analyze_user_ages_missing_key():
    """
    Tests the function with data that is missing an expected key.
    """
    users = [
        {"first": "John", "last": "Doe", "age": 30},
        {"last": "Smith", "age": 45},  # Missing 'first' name
        {"first": "Jane", "last": "Doe", "age": 25},
    ]
    # This test will crash before the assertion is ever reached.
    analyze_user_ages(users)
```

Now, let's run pytest and see what happens.

### Failure Demonstration: The `KeyError`

```bash
$ pytest test_analytics_v1.py
=========================== test session starts ============================
platform linux -- Python 3.11.4, pytest-7.4.0, pluggy-1.2.0
rootdir: /path/to/project
plugins: anyio-3.7.1
collected 1 item

test_analytics_v1.py F                                               [100%]

================================= FAILURES =================================
_____________________ test_analyze_user_ages_missing_key _____________________

    def test_analyze_user_ages_missing_key():
        """
        Tests the function with data that is missing an expected key.
        """
        users = [
            {"first": "John", "last": "Doe", "age": 30},
            {"last": "Smith", "age": 45},  # Missing 'first' name
            {"first": "Jane", "last": "Doe", "age": 25},
        ]
        # This test will crash before the assertion is ever reached.
>       analyze_user_ages(users)

test_analytics_v1.py:12:
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _

users = [{'first': 'John', 'last': 'Doe', 'age': 30}, {'last': 'Smith', 'age': 45}, {'first': 'Jane', 'last': 'Doe', 'age': 25}]

    def analyze_user_ages(users: list[dict]) -> dict:
        """
        Analyzes a list of user data, returning statistics.
        - Calculates average age.
        - Counts number of adults (age >= 18).
        """
        total_age = sum(user['age'] for user in users)
        num_users = len(users)
        average_age = total_age / num_users

        # Bug: Logic is incorrect for counting adults (should be >= 18)
        adult_count = 0
        for user in users:
            if user['age'] > 18:
                adult_count += 1

        # Bug: Assumes 'first' and 'last' keys will always exist
>       user_names = [f"{user['last']}, {user['first']}" for user in users]
E       KeyError: 'first'

user_analytics.py:19: KeyError
========================= short test summary info ==========================
FAILED test_analytics_v1.py::test_analyze_user_ages_missing_key - KeyError: 'first'
============================ 1 failed in 0.12s =============================
```

### Diagnostic Analysis: Reading the Failure

This output is dense, but it's a structured narrative of the error. Let's parse it section by section.

**1. The summary line**:
```
FAILED test_analytics_v1.py::test_analyze_user_ages_missing_key - KeyError: 'first'
```
-   **What this tells us**: The test named `test_analyze_user_ages_missing_key` in the file `test_analytics_v1.py` failed. The reason for the failure was a `KeyError` because the key `'first'` could not be found in a dictionary. This is our high-level summary.

**2. The traceback**:
```
_____________________ test_analyze_user_ages_missing_key _____________________

    def test_analyze_user_ages_missing_key():
...
>       analyze_user_ages(users)

test_analytics_v1.py:12:
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _

users = [...]

    def analyze_user_ages(users: list[dict]) -> dict:
...
>       user_names = [f"{user['last']}, {user['first']}" for user in users]
E       KeyError: 'first'

user_analytics.py:19: KeyError
```
-   **What this tells us**: This is the call stack, read from top to bottom. It shows the path the code took to arrive at the error.
-   The first block shows our test code. The `>` points to the exact line in our test file that triggered the error: `analyze_user_ages(users)`.
-   The second block takes us *inside* the `analyze_user_ages` function. The `>` here points to the line of code that actually crashed: `user_names = [f"{user['last']}, {user['first']}" for user in users]`.
-   **Key line**: The line marked with `E` (for Error) gives us the specific exception: `E       KeyError: 'first'`. This is the final, most specific piece of information.

**3. Assertion Introspection**:
-   In this case, there is no assertion introspection section. This is a crucial clue! The test didn't fail because an `assert` statement was false; it failed because the code crashed with an unhandled exception *before* it even reached an assertion.

**Root cause identified**: The function `analyze_user_ages` attempts to access the dictionary key `'first'` on every user dictionary, but the second user in our test data does not have this key.

**Why the current approach can't solve this**: Our code makes an unsafe assumption about the structure of its input data.

**What we need**: A way to handle potentially missing keys gracefully. For now, let's fix the bug so we can explore other failure modes. We'll use the `.get()` dictionary method, which allows providing a default value if a key is missing.

```python
# user_analytics.py (fixed version 1)

def analyze_user_ages(users: list[dict]) -> dict:
    """
    Analyzes a list of user data, returning statistics.
    - Calculates average age.
    - Counts number of adults (age >= 18).
    """
    total_age = sum(user['age'] for user in users)
    num_users = len(users)
    average_age = total_age / num_users

    # Bug: Logic is incorrect for counting adults (should be >= 18)
    adult_count = 0
    for user in users:
        if user['age'] > 18:
            adult_count += 1

    # FIX: Use .get() to handle missing keys gracefully
    user_names = [
        f"{user.get('last', 'N/A')}, {user.get('first', 'N/A')}" for user in users
    ]

    return {
        "average_age": average_age,
        "adult_count": adult_count,
        "user_names": sorted(user_names)
    }
```

With this fix, the `KeyError` is resolved. But our function still has other bugs. Reading the traceback is the first and most fundamental step in debugging any failing test.

### Limitation Preview

We've fixed a crash, but what about errors that are more subtle? Our function can still produce the wrong *answer* without crashing. These are often harder to find. To diagnose them, we'll need to see more detail about which tests are running and what their results are.

## Using pytest's Verbose and Extra-Verbose Modes

## Iteration 2: The Logical Error

Our code no longer crashes on malformed data, but does it produce the correct results? Let's write a new test that checks the logic of our `adult_count`. We'll include a user who is exactly 18 years old. According to our requirements, they should be counted as an adult.

```python
# test_analytics_v2.py
from user_analytics import analyze_user_ages

# NOTE: We are using the version of user_analytics with the KeyError fix.

def test_analyze_user_ages_adult_count_logic():
    """
    Tests that the adult count logic is correct, especially for the edge case of age 18.
    """
    users = [
        {"first": "John", "last": "Doe", "age": 30},    # Adult
        {"first": "Jane", "last": "Smith", "age": 17},  # Minor
        {"first": "Sam", "last": "Jones", "age": 18},   # Adult (edge case)
    ]
    result = analyze_user_ages(users)
    # The bug is `> 18` instead of `>= 18`, so this will fail.
    # Expected adult_count is 2, but the function will return 1.
    assert result["adult_count"] == 2
```

### Failure Demonstration: The Silent Logic Bug

Let's run this new test.

```bash
$ pytest test_analytics_v2.py
=========================== test session starts ============================
platform linux -- Python 3.11.4, pytest-7.4.0, pluggy-1.2.0
rootdir: /path/to/project
collected 1 item

test_analytics_v2.py F                                               [100%]

================================= FAILURES =================================
__________________ test_analyze_user_ages_adult_count_logic __________________

    def test_analyze_user_ages_adult_count_logic():
        """
        Tests that the adult count logic is correct, especially for the edge case of age 18.
        """
        users = [
            {"first": "John", "last": "Doe", "age": 30},    # Adult
            {"first": "Jane", "last": "Smith", "age": 17},  # Minor
            {"first": "Sam", "last": "Jones", "age": 18},   # Adult (edge case)
        ]
        result = analyze_user_ages(users)
        # The bug is `> 18` instead of `>= 18`, so this will fail.
        # Expected adult_count is 2, but the function will return 1.
>       assert result["adult_count"] == 2
E       assert 1 == 2
E        +  where 1 = {'average_age': 21.666666666666668, 'adult_count': 1, 'user_names': ['Doe, John', 'Jones, Sam', 'Smith, Jane']}['adult_count']

test_analytics_v2.py:15: AssertionError
========================= short test summary info ==========================
FAILED test_analytics_v2.py::test_analyze_user_ages_adult_count_logic - assert 1 == 2
============================ 1 failed in 0.12s =============================
```

### Diagnostic Analysis: A Failed Assertion

This failure output looks different from the last one.

1.  **The summary line**: `FAILED ... - assert 1 == 2`. It tells us the failure was an `AssertionError`.
2.  **The traceback**: It points directly to the `assert` line in our test.
3.  **The assertion introspection**: This is the new, critical section.
    ```
    >       assert result["adult_count"] == 2
    E       assert 1 == 2
    E        +  where 1 = {'average_age': ..., 'adult_count': 1, ...}['adult_count']
    ```
    -   Pytest re-writes the `assert` statement to show us the *actual values* involved at the time of failure.
    -   `assert 1 == 2` is the simplified comparison. It's immediately clear what went wrong.
    -   The `where` clause is even more helpful. It shows us that the `1` came from `result["adult_count"]`, and it even shows us the *entire* `result` dictionary. This is incredibly powerful context.

**Root cause identified**: The function returned `adult_count: 1` when we expected `2`. The user aged 18 was not counted.

### Technique Introduced: Verbose (`-v`) and Extra-Verbose (`-vv`)

The default output is great for failures, but when you have many tests, the progress indicator (`.F.s.`) can be terse. The `-v` (verbose) flag gives more readable output.

```bash
$ pytest -v test_analytics_v2.py
=========================== test session starts ============================
platform linux -- Python 3.11.4, pytest-7.4.0, pluggy-1.2.0
rootdir: /path/to/project
collected 1 item

test_analytics_v2.py::test_analyze_user_ages_adult_count_logic FAILED [100%]

================================= FAILURES =================================
__________________ test_analyze_user_ages_adult_count_logic __________________
... (failure details are the same) ...
========================= short test summary info ==========================
FAILED test_analytics_v2.py::test_analyze_user_ages_adult_count_logic - assert 1 == 2
============================ 1 failed in 0.12s =============================
```

The key difference is in the progress report:
-   **Default**: `test_analytics_v2.py F`
-   **Verbose (`-v`)**: `test_analytics_v2.py::test_analyze_user_ages_adult_count_logic FAILED`

With `-v`, pytest prints the full node ID of each test and its status (`PASSED`, `FAILED`, `SKIPPED`), which is much easier to read, especially in a large test suite.

The `-vv` (extra-verbose) flag adds even more detail, including the docstrings of your test functions. This is useful if you follow a convention of writing detailed explanations in your docstrings.

### When to Apply This Solution
-   **What it optimizes for**: Readability of the test run, especially the progress and summary sections.
-   **When to choose this approach**: Almost always use `-v` for local development. It provides a better signal-to-noise ratio than the default dot output. Use `-vv` if your docstrings contain critical information for understanding a test's purpose.
-   **When to avoid this approach**: In CI systems where logs are already verbose, the default output might be preferred to keep logs concise.

Let's apply the fix for the logical error.

```python
# user_analytics.py (fixed version 2)
...
    # FIX: Logic is now correct for counting adults (>= 18)
    adult_count = 0
    for user in users:
        if user['age'] >= 18: # Changed from > to >=
            adult_count += 1
...
```

### Limitation Preview

Verbose mode helps us see *what* failed more clearly, but when we have a cascade of failures, the sheer volume of output can be overwhelming. We need a way to focus on one problem at a time.

## The -x Flag (Stop on First Failure)

## Iteration 3: The Error Cascade

In a real project, a single bug can cause many tests to fail. This creates a lot of noise in the test report, making it hard to know where to start debugging.

Let's create a test file that will trigger multiple failures in our original buggy code. We will revert `user_analytics.py` to its initial state (with both the `KeyError` and the logic bug) and add a new test for a `ZeroDivisionError`.

```python
# user_analytics.py (reverted to initial buggy version)

def analyze_user_ages(users: list[dict]) -> dict:
    total_age = sum(user['age'] for user in users)
    num_users = len(users)
    average_age = total_age / num_users # Potential ZeroDivisionError

    adult_count = 0
    for user in users:
        if user['age'] > 18: # Bug 1: Incorrect logic
            adult_count += 1

    user_names = [f"{user['last']}, {user['first']}" for user in users] # Bug 2: Potential KeyError

    return {
        "average_age": average_age,
        "adult_count": adult_count,
        "user_names": sorted(user_names)
    }
```

```python
# test_analytics_v3.py
from user_analytics import analyze_user_ages

def test_logic_error_adult_count():
    """Triggers the adult count logic error."""
    users = [{"first": "Sam", "last": "Jones", "age": 18}]
    result = analyze_user_ages(users)
    assert result["adult_count"] == 1

def test_crash_on_missing_key():
    """Triggers the KeyError."""
    users = [{"last": "Smith", "age": 45}]
    analyze_user_ages(users)

def test_crash_on_empty_list():
    """Triggers the ZeroDivisionError."""
    users = []
    analyze_user_ages(users)
```

### Failure Demonstration: Too Much Information

Running this file produces a long, intimidating report with three distinct failures.

```bash
$ pytest -v test_analytics_v3.py
=========================== test session starts ============================
...
collected 3 items

test_analytics_v3.py::test_logic_error_adult_count FAILED             [ 33%]
test_analytics_v3.py::test_crash_on_missing_key FAILED                [ 66%]
test_analytics_v3.py::test_crash_on_empty_list FAILED                 [100%]

================================= FAILURES =================================
_______________________ test_logic_error_adult_count _______________________
... (AssertionError details) ...
________________________ test_crash_on_missing_key _________________________
... (KeyError details) ...
________________________ test_crash_on_empty_list __________________________
... (ZeroDivisionError details) ...
========================= short test summary info ==========================
FAILED test_analytics_v3.py::test_logic_error_adult_count - assert 0 == 1
FAILED test_analytics_v3.py::test_crash_on_missing_key - KeyError: 'first'
FAILED test_analytics_v3.py::test_crash_on_empty_list - ZeroDivisionError: division by zero
============================ 3 failed in 0.13s =============================
```

### Diagnostic Analysis

When faced with multiple failures, the best strategy is often to **fix the first one and rerun**. A single root cause can trigger failures in seemingly unrelated tests. The noise from subsequent failures can distract from the real issue.

**Root cause identified**: We have at least three separate bugs.
**Why the current approach can't solve this**: Looking at all three failures at once is inefficient. We need to focus our attention.

### Technique Introduced: `-x` (Stop on First Failure)

Pytest provides the `-x` flag (or its long-form alias `--exitfirst`) to address this exact problem. When you run pytest with `-x`, the test session will stop immediately after the first test fails.

```bash
$ pytest -v -x test_analytics_v3.py
=========================== test session starts ============================
...
collected 3 items

test_analytics_v3.py::test_logic_error_adult_count FAILED             [ 33%]

================================= FAILURES =================================
_______________________ test_logic_error_adult_count _______________________

    def test_logic_error_adult_count():
        """Triggers the adult count logic error."""
        users = [{"first": "Sam", "last": "Jones", "age": 18}]
        result = analyze_user_ages(users)
>       assert result["adult_count"] == 1
E       assert 0 == 1
E        +  where 0 = {'average_age': 18.0, 'adult_count': 0, 'user_names': ['Jones, Sam']}['adult_count']

test_analytics_v3.py:7: AssertionError
================== stopping after 1 failures ===================
========================= short test summary info ==========================
FAILED test_analytics_v3.py::test_logic_error_adult_count - assert 0 == 1
======================= 1 failed, 2 deselected in 0.12s ====================
```

### Expected vs. Actual Improvement

The output is now much cleaner.
-   Pytest ran the first test, `test_logic_error_adult_count`, which failed.
-   It immediately stopped the session, as indicated by the `stopping after 1 failures` message.
-   The summary shows that one test failed and two were `deselected` (meaning they were discovered but never run).

This allows us to focus all our attention on fixing the first bug. Once it's fixed, we can rerun the suite and see what the *next* failure is. This creates a methodical, one-at-a-time debugging workflow.

### When to Apply This Solution
-   **What it optimizes for**: Developer focus and a clean signal during debugging.
-   **When to choose this approach**:
    -   During local development, when you've made a change that caused many tests to fail.
    -   In a CI/CD pipeline to get faster feedback. If a fundamental test fails, there's no point in wasting time and resources running hundreds of other tests that are also likely to fail.
-   **When to avoid this approach**: When you are trying to get a complete picture of the health of your test suite and want to see *all* current failures.

### Limitation Preview

Stopping at the first failure is great for focus, but reading the traceback only tells us what happened *after the fact*. What if we need to inspect the program's state—the values of all the variables—at the exact moment of the crash? For that, we need an interactive debugger.

## The --pdb Flag (Drop into Debugger)

## Iteration 4: Post-Mortem Debugging

So far, we've been analyzing static reports. The `--pdb` flag transforms a test failure from a static report into a live, interactive investigation. When a test fails with an exception or a failed assertion, `--pdb` automatically drops you into the Python Debugger (`pdb`) at the exact point of failure.

Let's focus on the `ZeroDivisionError` from our test suite.

```python
# test_analytics_v4.py
from user_analytics import analyze_user_ages

# Using the buggy version of analyze_user_ages

def test_crash_on_empty_list():
    """Triggers the ZeroDivisionError."""
    users = []
    analyze_user_ages(users)
```

### Failure Demonstration: Dropping into PDB

We'll run this single test with the `--pdb` flag.

```bash
$ pytest --pdb test_analytics_v4.py
=========================== test session starts ============================
...
collected 1 item

test_analytics_v4.py F                                               [100%]

================================= FAILURES =================================
________________________ test_crash_on_empty_list __________________________

    def test_crash_on_empty_list():
        """Triggers the ZeroDivisionError."""
        users = []
>       analyze_user_ages(users)

test_analytics_v4.py:8:
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
user_analytics.py:6: in analyze_user_ages
    average_age = total_age / num_users
E   ZeroDivisionError: division by zero

>>>>>>>>>>>>>>>>>>>>>>>>> entering PDB >>>>>>>>>>>>>>>>>>>>>>>>>
> /path/to/project/user_analytics.py(6)analyze_user_ages()
-> average_age = total_age / num_users
(Pdb)
```

### Diagnostic Analysis: An Interactive Session

Notice that the test run has paused. The `(Pdb)` prompt indicates we are now in an interactive debugging session, stopped on the very line that caused the `ZeroDivisionError`.

We can now inspect the state of the program. Here are the most common `pdb` commands:
-   `p <expression>`: Print the value of an expression.
-   `pp <expression>`: Pretty-print the value of an expression.
-   `l` (list): Show the source code around the current line.
-   `w` (where): Print the full traceback or call stack.
-   `up` / `down`: Move up or down the call stack to inspect variables in other functions.
-   `c` (continue): Resume execution.
-   `q` (quit): Exit the debugger and the test session.

Let's use these commands to investigate our crash.

```bash
(Pdb) p num_users
0
(Pdb) p total_age
0
(Pdb) p users
[]
```
Instantly, we see the problem. We are trying to divide by `num_users`, which is `0`.

Now let's move up the call stack to see where the empty list came from.

```bash
(Pdb) w
  /path/to/project/test_analytics_v4.py(8)test_crash_on_empty_list()
-> analyze_user_ages(users)
> /path/to/project/user_analytics.py(6)analyze_user_ages()
-> average_age = total_age / num_users
(Pdb) up
> /path/to/project/test_analytics_v4.py(8)test_crash_on_empty_list()
-> analyze_user_ages(users)
(Pdb) p users
[]
```
By using `up`, we've moved from the `analyze_user_ages` function back into our test function, `test_crash_on_empty_list`. Printing `users` here confirms that we passed an empty list into the function, which is the ultimate source of the error.

**Root cause identified**: The function does not handle an empty `users` list, leading to division by zero.

### When to Apply This Solution
-   **What it optimizes for**: Deep, interactive inspection of program state at the moment of failure.
-   **When to choose this approach**: When a traceback isn't enough to understand *why* a variable has a certain value. It's the go-to tool for complex bugs that cause crashes or assertion failures.
-   **When to avoid this approach**: In non-interactive environments like a CI/CD pipeline. This tool is for local, hands-on debugging.

### Limitation Preview

The `--pdb` flag is a "post-mortem" tool—it only activates *after* an error has occurred. But what if a test fails a simple assertion, and the problem actually happened much earlier, like inside a loop? We need a way to pause execution *before* the failure, at a location of our choosing.

## Using Breakpoints in Tests

## Iteration 5: Proactive Debugging with `breakpoint()`

Sometimes, the line that fails is not the line with the bug. The bug might be a subtle logic error inside a loop that corrupts a value, with the assertion failure only happening much later. For these cases, we need to set a trap—a breakpoint—to pause execution at a specific line we want to investigate.

Let's return to our `adult_count` logical error. The assertion `assert result["adult_count"] == 1` fails, but the real bug is inside the `for` loop in the `analyze_user_ages` function. Using `--pdb` would just drop us at the `assert` line, which is too late. We want to see what's happening *inside the loop*.

### Technique Introduced: `breakpoint()`

Since Python 3.7, the built-in `breakpoint()` function is the standard way to set a debugger breakpoint. (In older versions, you would use `import pdb; pdb.set_trace()`). When the Python interpreter encounters this function, it pauses execution and starts a `pdb` session, just like `--pdb`, but at the exact location you specified.

Let's add a breakpoint to our application code to inspect the loop.

```python
# user_analytics.py (with breakpoint)

def analyze_user_ages(users: list[dict]) -> dict:
    ...
    adult_count = 0
    for user in users:
        breakpoint() # Pause execution here on every iteration
        if user['age'] > 18:
            adult_count += 1
    ...
    return { ... }
```

```python
# test_analytics_v5.py
from user_analytics import analyze_user_ages

def test_logic_error_with_breakpoint():
    """
    Uses a breakpoint in the source code to debug the adult count logic.
    """
    users = [
        {"first": "Jane", "last": "Smith", "age": 17},
        {"first": "Sam", "last": "Jones", "age": 18},
    ]
    result = analyze_user_ages(users)
    assert result["adult_count"] == 1
```

### Failure Demonstration: Stepping Through Code

Now, run the test. Pytest will start the test, and execution will pause as soon as it hits our `breakpoint()`.

```bash
$ pytest test_analytics_v5.py
=========================== test session starts ============================
...
collected 1 item

test_analytics_v5.py
>>>>>>>>>>>>>>>>>>>>>>>>> PDB set_trace >>>>>>>>>>>>>>>>>>>>>>>>>
> /path/to/project/user_analytics.py(13)analyze_user_ages()
-> if user['age'] > 18:
(Pdb)
```

We are now paused at the beginning of the first loop iteration. Let's inspect the state.

```bash
(Pdb) p user
{'first': 'Jane', 'last': 'Smith', 'age': 17}
(Pdb) p adult_count
0
```
This looks correct. The user is 17, and the count is 0. Let's type `c` (continue) to proceed to the next iteration. The code will run through the loop once and stop at the `breakpoint()` again.

```bash
(Pdb) c
> /path/to/project/user_analytics.py(13)analyze_user_ages()
-> if user['age'] > 18:
(Pdb)
```
Now we're in the second iteration. Let's inspect again.

```bash
(Pdb) p user
{'first': 'Sam', 'last': 'Jones', 'age': 18}
(Pdb) p adult_count
0
```
Here is our "Aha!" moment. The current user is 18, but `adult_count` is still 0 from the previous iteration. We can now execute the *next* line of code using the `n` (next) command to see what happens.

```bash
(Pdb) n
> /path/to/project/user_analytics.py(14)analyze_user_ages()
-> adult_count += 1
(Pdb)
```
Wait, that's not right. The debugger prompt shows that the next line to be executed is `adult_count += 1`. But the condition `if user['age'] > 18` should have been false! Let's re-check the condition ourselves.

```bash
(Pdb) p user['age'] > 18
False
```
Ah, the debugger stepped *over* the `if` block because the condition was false. This is the moment we realize our logic is wrong. The condition `user['age'] > 18` evaluates to `False` for the 18-year-old user, so `adult_count` is never incremented. The logic should have been `user['age'] >= 18`.

**Root cause identified**: By stepping through the code line-by-line and inspecting variables, we proved that our conditional logic was incorrect for the edge case.

### Limitation Preview

Interactive debugging is the most powerful tool in our arsenal, but it can be slow and cumbersome. For simpler problems, or to get a high-level overview of a function's execution, it's often overkill. Sometimes, all we need is a simple log of what happened, without having to manually step through the code.

## Logging and Debugging Information

## Iteration 6: Tracing Execution with Logs and Prints

The final debugging technique is often the simplest: printing or logging information as your code executes. This is less interactive than a debugger but provides a quick and easy way to trace a program's flow and inspect variable values at different points in time.

Pytest has special handling for both `print()` statements and Python's built-in `logging` module.

### Technique Introduced: Capturing Output

By default, pytest captures any output sent to `stdout` (like from `print()`) and `stderr`. It will only display this captured output for *failing* tests. This keeps your test runs clean when everything is passing.

Let's add some `print` statements to our function to see what's going on.

```python
# user_analytics.py (with print statements)

def analyze_user_ages(users: list[dict]) -> dict:
    print(f"\nAnalyzing {len(users)} users...")
    ...
    adult_count = 0
    for user in users:
        is_adult = user['age'] >= 18
        print(f"  - Processing {user.get('first', 'N/A')}: age {user['age']}, is_adult: {is_adult}")
        if is_adult:
            adult_count += 1
    ...
    print(f"Analysis complete. Found {adult_count} adults.")
    return { ... }
```

Now, let's rerun our failing test for the logic error (`test_analytics_v5.py`).

```bash
$ pytest test_analytics_v5.py
=========================== test session starts ============================
...
collected 1 item

test_analytics_v5.py F                                               [100%]

================================= FAILURES =================================
______________________ test_logic_error_with_breakpoint ______________________
... (traceback and assertion introspection) ...
-------------------------- Captured stdout call ---------------------------
Analyzing 2 users...
  - Processing Jane: age 17, is_adult: False
  - Processing Sam: age 18, is_adult: True
Analysis complete. Found 1 adults.
========================= short test summary info ==========================
...
```

The output from our `print` statements appears in a dedicated `Captured stdout call` section for the failing test. This log clearly shows the intermediate state and can often be enough to spot the bug without a full debugger session.

### Displaying Output for Passing Tests with `-s`

If you want to see the output even for passing tests, you can use the `-s` flag (or `--capture=no`). This disables all output capturing and prints directly to the console.

```bash
$ pytest -s test_analytics_v5.py # Assuming the test now passes
=========================== test session starts ============================
...
collected 1 item

test_analytics_v5.py
Analyzing 2 users...
  - Processing Jane: age 17, is_adult: False
  - Processing Sam: age 18, is_adult: True
Analysis complete. Found 1 adults.
.                                                                    [100%]

============================ 1 passed in 0.01s =============================
```

### Using the `logging` Module

While `print` is easy, Python's `logging` module is more powerful and configurable. Pytest also captures log messages and can display them based on a specified log level.

To use it, you configure a logger and use it instead of `print`.

```python
# user_analytics.py (with logging)
import logging

log = logging.getLogger(__name__)

def analyze_user_ages(users: list[dict]) -> dict:
    log.info(f"Analyzing {len(users)} users...")
    ...
    for user in users:
        is_adult = user['age'] >= 18
        log.debug(f"  - Processing {user.get('first', 'N/A')}: age {user['age']}, is_adult: {is_adult}")
        ...
    log.info(f"Analysis complete. Found {adult_count} adults.")
    return { ... }
```

To see these logs, you need to tell pytest the minimum level to display.

```bash
# Show logs at INFO level and above
$ pytest --log-cli-level=INFO

# Show all logs, including DEBUG level
$ pytest --log-cli-level=DEBUG
```

This will produce a `Captured log call` section in the report for failing tests, similar to the one for `stdout`.

### When to Apply This Solution
-   **`print` with `-s`**: Best for quick, temporary debugging. It's simple and effective for tracing values.
-   **`logging` with `--log-cli-level`**: A more robust, permanent solution. It allows you to have different levels of verbosity (e.g., `DEBUG`, `INFO`, `WARNING`) and to enable them in tests without modifying the application code. This is the preferred method for libraries or long-lived applications.

## Synthesis: The Complete Journey

We started with a buggy function and systematically used pytest's debugging tools to find and fix three distinct types of errors: a crash (`KeyError`), a silent logical error, and another crash (`ZeroDivisionError`).

### The Journey: From Problem to Solution

| Iteration | Failure Mode              | Technique Applied                               | Result                                                              |
| :-------- | :------------------------ | :---------------------------------------------- | :------------------------------------------------------------------ |
| 1         | `KeyError` crash          | Reading the traceback                           | Identified the exact line causing the crash due to missing data.    |
| 2         | Incorrect `adult_count`   | Assertion introspection & `-v`                  | Pinpointed a logical error by comparing expected vs. actual values. |
| 3         | Multiple cascading errors | `-x` (stop on first failure)                    | Focused the debugging effort by isolating the first failure.        |
| 4         | `ZeroDivisionError` crash | `--pdb` (post-mortem debugger)                  | Interactively inspected variables at the crash site to find cause.  |
| 5         | Hard-to-find logic error  | `breakpoint()` (proactive debugging)            | Paused execution inside a loop to observe state changes over time.  |
| 6         | Need for execution trace  | `print()`/`logging` with pytest capture options | Generated a persistent trace of the function's execution path.      |

### Final Implementation

Here is the final, corrected version of our function, ready for production.

```python
# user_analytics.py (final version)
import logging

log = logging.getLogger(__name__)

def analyze_user_ages(users: list[dict]) -> dict:
    """
    Analyzes a list of user data, returning statistics.
    - Calculates average age.
    - Counts number of adults (age >= 18).
    """
    num_users = len(users)
    if not num_users:
        log.warning("analyze_user_ages called with an empty list.")
        return {"average_age": 0, "adult_count": 0, "user_names": []}

    total_age = sum(user.get('age', 0) for user in users)
    average_age = total_age / num_users

    adult_count = 0
    for user in users:
        if user.get('age', 0) >= 18:
            adult_count += 1

    user_names = [
        f"{user.get('last', 'N/A')}, {user.get('first', 'N/A')}" for user in users
    ]

    return {
        "average_age": average_age,
        "adult_count": adult_count,
        "user_names": sorted(user_names)
    }
```

### Decision Framework: Which Debugging Tool When?

When a test fails, use this mental flowchart to choose the right tool for the job.

1.  **Start Here: A test fails.**
    -   **Action**: Read the full pytest output carefully.
    -   **Question**: Is the problem obvious from the traceback and assertion introspection?
        -   **Yes**: Fix the bug. You're done!
        -   **No**: Proceed to step 2.

2.  **Problem: The output is overwhelming with many failures.**
    -   **Tool**: `pytest -x`
    -   **Action**: Rerun tests to stop at the very first failure. Focus on fixing that one.

3.  **Problem: The test crashes with an exception, and the traceback isn't enough to understand why.**
    -   **Tool**: `pytest --pdb`
    -   **Action**: Rerun the test. At the `(Pdb)` prompt, inspect variables (`p var_name`) to understand the program's state at the moment of the crash.

4.  **Problem: The test fails an assertion (the result is wrong), but doesn't crash.**
    -   **Tool**: `breakpoint()`
    -   **Action**: Add `breakpoint()` to your application code just before the section you suspect is buggy. Rerun the test and step through the code (`n`), inspecting variables (`p var_name`) along the way.

5.  **Problem: You just need a quick trace of values without an interactive session.**
    -   **Tool**: `print()` statements and `pytest -s`.
    -   **Action**: Add `print()` calls at key points in your code. Rerun with `-s` to see the output. This is great for "printf debugging."

6.  **Problem: You want to add permanent, configurable tracing to your application.**
    -   **Tool**: Python's `logging` module and `pytest --log-cli-level=...`.
    -   **Action**: Add `log.info()` or `log.debug()` calls. Control their visibility in tests using pytest's command-line flags.

By mastering this toolkit, you can systematically and efficiently diagnose any test failure, turning bugs from frustrating mysteries into solvable puzzles.
