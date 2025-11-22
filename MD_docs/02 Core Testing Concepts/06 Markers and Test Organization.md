# Chapter 6: Markers and Test Organization

## What Are Markers?

## What Are Markers?

As your test suite grows, you'll find that not all tests are created equal. Some are fast unit tests, others are slow integration tests. Some require a database connection, while others should only run on a specific operating system. How do you manage this complexity?

This is the problem that markers solve. **Markers are metadata you can apply to your test functions.** Think of them as tags or labels. You can "tag" a test as `slow`, `database`, or `smoke_test`, and then instruct pytest to run—or not run—tests based on these tags.

Markers are implemented as Python decorators, and their syntax is simple and readable:

```python
import pytest

@pytest.mark.slow
def test_very_long_computation():
    # ... code that takes a long time
    pass
```

In this example, we've marked `test_very_long_computation` with the label `slow`. This marker doesn't change how the test runs on its own, but it gives us a powerful handle to control it from the command line.

Markers are the primary tool in pytest for categorizing, filtering, and organizing your tests beyond the simple structure of files and directories. They allow you to create logical groupings that are essential for managing a large, real-world test suite.

## Built-in Markers (skip, xfail, filterwarnings)

## Built-in Markers (skip, xfail, filterwarnings)

Pytest comes with several useful markers out of the box. Let's explore the most common ones by first seeing the problems they solve.

### `skip`: Skipping a Test Conditionally

Imagine you're developing a new feature. You've written a test for it, but the feature's code isn't complete yet. If you run your test suite, this test will fail, creating noise and potentially breaking your continuous integration (CI) build.

**The Wrong Way: Commenting Out the Test**

You might be tempted to just comment out the test.

```python
# tests/test_feature.py

# def test_new_feature():
#     assert new_feature_function() == "expected result"
#     # This test will be forgotten!
```

The danger here is that commented-out code is easily forgotten. Months later, you might not remember why it's there or if it's still relevant.

**The Right Way: Using `@pytest.mark.skip`**

A much better approach is to explicitly mark the test to be skipped. This documents the intent and keeps the test visible in your test reports.

```python
# tests/test_skipping.py
import pytest
import sys

def test_always_passes():
    assert True

@pytest.mark.skip(reason="Feature not yet implemented.")
def test_new_feature():
    # This code will not be executed
    assert False

@pytest.mark.skipif(sys.version_info < (3, 10), reason="Requires Python 3.10 or higher")
def test_python_310_feature():
    # This test will only run on Python 3.10+
    # For example, using the 'match' statement
    value = 42
    match value:
        case 42:
            result = True
        case _:
            result = False
    assert result
```

Let's run this and see the output.

```bash
$ pytest -v tests/test_skipping.py
=========================== test session starts ============================
...
collected 3 items

tests/test_skipping.py::test_always_passes PASSED                      [ 33%]
tests/test_skipping.py::test_new_feature SKIPPED (Feature not yet implemented.) [ 66%]
tests/test_skipping.py::test_python_310_feature SKIPPED (Requires Python 3.10 or higher) [100%]
# Note: The second SKIPPED message will vary based on your Python version.

====================== 1 passed, 2 skipped in 0.01s ======================
```

Notice the output:
- `PASSED`: The test ran and succeeded.
- `SKIPPED`: Pytest recognized the marker and did not execute the test function.
- The `reason` string is printed in the test summary, which is crucial for understanding why a test was skipped.

There are two variants:
1.  `@pytest.mark.skip(reason="...")`: Always skips the test.
2.  `@pytest.mark.skipif(condition, reason="...")`: Skips the test only if the `condition` evaluates to `True`. This is perfect for tests that are platform-specific or depend on a certain library version.

### `xfail`: Marking a Test as an Expected Failure

Now consider a different scenario: you've found a bug in your code. You write a test that reproduces the bug. This test is valuable! It proves the bug exists and will signal when it's fixed. However, until the bug is fixed, this test will fail your test suite.

You don't want to `skip` it, because you *want* it to run. You just want to tell pytest, "I know this is broken, and that's okay for now." This is the job of `xfail` (expected failure).

```python
# tests/test_xfail.py
import pytest

# A buggy function we need to fix
def buggy_function():
    return False

@pytest.mark.xfail(reason="Bug #123: buggy_function returns False")
def test_known_bug():
    assert buggy_function() is True

def test_another_feature():
    assert True
```

When we run this, pytest executes `test_known_bug`, sees that it fails as expected, and marks it accordingly.

```bash
$ pytest -v tests/test_xfail.py
=========================== test session starts ============================
...
collected 2 items

tests/test_xfail.py::test_known_bug XFAIL (Bug #123: buggy_function returns False) [ 50%]
tests/test_xfail.py::test_another_feature PASSED                       [100%]

====================== 1 passed, 1 xfailed in 0.01s ======================
```

The test suite passes, but the report clearly shows one test is an `XFAIL`. This is the ideal way to manage tests for known bugs.

What happens if we fix the bug? Let's see.

```python
# tests/test_xpass.py
import pytest

# The function is now fixed!
def formerly_buggy_function():
    return True

@pytest.mark.xfail(reason="Bug #123: This should be fixed now")
def test_unexpected_pass():
    assert formerly_buggy_function() is True
```

Now, when we run the test, something interesting happens. The test runs, it passes, but pytest knows we *expected* it to fail. This is called an `XPASS` (expected failure, but it passed).

```bash
$ pytest -v tests/test_xpass.py
=========================== test session starts ============================
...
collected 1 item

tests/test_xpass.py::test_unexpected_pass XPASS (Bug #123: This should be fixed now) [100%]

====================== 1 xpassed in 0.02s ======================
```

By default, an `XPASS` does not fail the test suite. However, it's a strong signal that the underlying bug has been fixed and you should remove the `@pytest.mark.xfail` marker. This prevents you from having outdated markers in your code.

### `filterwarnings`: Suppressing Known Warnings

Sometimes, your code or its dependencies will issue warnings (e.g., `DeprecationWarning`). While these don't cause tests to fail, they can clutter your test output and hide important information.

Let's imagine a function that uses a deprecated feature.

```python
# tests/test_warnings.py
import pytest
import warnings

def function_with_deprecation():
    warnings.warn("This function is deprecated", DeprecationWarning)
    return 42

def test_deprecation_normally():
    assert function_with_deprecation() == 42

@pytest.mark.filterwarnings("ignore:This function is deprecated")
def test_deprecation_with_filter():
    assert function_with_deprecation() == 42
```

Running this file shows the difference clearly.

```bash
$ pytest -v tests/test_warnings.py
=========================== test session starts ============================
...
collected 2 items

tests/test_warnings.py::test_deprecation_normally PASSED               [ 50%]
tests/test_warnings.py::test_deprecation_with_filter PASSED            [100%]

============================== Warnings ==================================
tests/test_warnings.py::test_deprecation_normally
  /path/to/tests/test_warnings.py:6: DeprecationWarning: This function is deprecated
    warnings.warn("This function is deprecated", DeprecationWarning)

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
====================== 2 passed, 1 warning in 0.02s ======================
```

As you can see, the first test ran successfully but generated a warning in the final report. The second test, decorated with `filterwarnings`, ran without producing any warning output. This is useful for temporarily silencing warnings from third-party libraries or legacy code that you don't intend to fix immediately.

## Creating Custom Markers

## Creating Custom Markers

While the built-in markers are useful, the real power comes from creating your own. This allows you to categorize your tests according to the logic of your own application.

Let's use a common scenario. In a typical project, you might have:
-   **Unit tests**: Fast, isolated, test a single piece of logic.
-   **Integration tests**: Slower, test how multiple components work together.
-   **API tests**: Even slower, make real network requests to an external service.

Creating custom markers is as simple as using them. You don't need to declare them anywhere first (though we'll see why you *should* in the next section).

Let's write a test file that uses these categories.

```python
# tests/test_categories.py
import pytest
import time

@pytest.mark.unit
def test_sum_numbers():
    assert 2 + 2 == 4

@pytest.mark.integration
def test_db_connection():
    # Pretend this connects to a database
    time.sleep(0.5)
    assert True

@pytest.mark.api
@pytest.mark.slow
def test_api_call():
    # Pretend this makes a network call
    time.sleep(1)
    assert True
```

That's it! We've just "created" three new markers: `unit`, `integration`, and `api`. We also added the `slow` marker to the API test, demonstrating that you can apply multiple markers to a single test.

Right now, these markers don't do anything by themselves. They are just metadata waiting to be used. Their power is unlocked when we use them for filtering, which we'll cover in section 6.5. But first, let's address a small problem this code creates.

## Registering Markers in Configuration

## Registering Markers in Configuration

If you run the test file from the previous section, you'll notice something new in the output.

```bash
$ pytest tests/test_categories.py
=========================== test session starts ============================
...
collected 3 items

tests/test_categories.py ...                                         [100%]

============================== Warnings ==================================
.../tests/test_categories.py:5
  PytestUnknownMarkWarning: Unknown pytest.mark.unit - is this a typo?
    @pytest.mark.unit

.../tests/test_categories.py:9
  PytestUnknownMarkWarning: Unknown pytest.mark.integration - is this a typo?
    @pytest.mark.integration

.../tests/test_categories.py:14
  PytestUnknownMarkWarning: Unknown pytest.mark.api - is this a typo?
    @pytest.mark.api

-- Docs: https://docs.pytest.org/en/stable/how-to/mark.html
====================== 3 passed, 3 warnings in 1.52s =====================
```

Pytest runs the tests, but it issues a `PytestUnknownMarkWarning` for each of our custom markers. Why?

This is a safety feature. Pytest warns you about unrecognized markers to help you catch typos. If you accidentally typed `@pytest.mark.integraton` instead of `@pytest.mark.integration`, you would want pytest to tell you! Without this warning, your misspelled marker would be silently ignored, and the test wouldn't be included in the `integration` group.

To make pytest recognize our markers—and to document them for other developers—we should register them in a configuration file. The most common place for this is `pytest.ini` in the root of your project.

Let's create that file.

```ini
# pytest.ini
[pytest]
markers =
    unit: marks tests as unit tests (fast, isolated)
    integration: marks tests as integration tests (slower, may require services)
    api: marks tests as API tests (slowest, requires network access)
    slow: marks tests as slow to run
```

The format is simple: under the `[pytest]` section, add a `markers` key. Each line that follows is a marker name, a colon, and a description. The description is excellent documentation for your team.

Now, with `pytest.ini` in our project root, let's run the tests again.

```bash
$ pytest tests/test_categories.py
=========================== test session starts ============================
...
collected 3 items

tests/test_categories.py ...                                         [100%]

========================= 3 passed in 1.52s ==========================
```

The warnings are gone! We have officially told pytest about our custom markers. This is a critical best practice for any project that uses custom markers. You can also see a list of all available markers (including built-in ones) by running `pytest --markers`.

## Filtering Tests by Markers

## Filtering Tests by Markers

Now we get to the payoff. We've tagged our tests with metadata; it's time to use that metadata to control which tests are run. This is done with the `-m` command-line option.

Let's use our `tests/test_categories.py` file from before.

### Running a Single Group of Tests

To run only the unit tests, we use `-m unit`.

```bash
$ pytest -v -m unit
=========================== test session starts ============================
...
collected 3 items / 2 deselected / 1 selected

tests/test_categories.py::test_sum_numbers PASSED                      [100%]

=================== 1 passed, 2 deselected in 0.01s ====================
```

Pytest reports that it collected 3 tests but deselected 2, running only the 1 test marked with `unit`.

### Excluding a Group of Tests

Perhaps more commonly, you'll want to run your fast tests and exclude the slow ones. You can do this with `not`.

```bash
$ pytest -v -m "not slow"
=========================== test session starts ============================
...
collected 3 items / 1 deselected / 2 selected

tests/test_categories.py::test_sum_numbers PASSED                      [ 50%]
tests/test_categories.py::test_db_connection PASSED                    [100%]

=================== 2 passed, 1 deselected in 0.51s ====================
```

Notice the quotes around `"not slow"`. They are often necessary because command-line shells can interpret characters like `(` or `)` as special commands. It's a good habit to always quote your `-m` expression.

### Combining Markers with `and` and `or`

You can build more complex queries using boolean logic.

-   **`and`**: Run tests that have *all* the specified markers.
-   **`or`**: Run tests that have *any* of the specified markers.

Let's run tests that are marked as both `api` AND `slow`.

```bash
$ pytest -v -m "api and slow"
=========================== test session starts ============================
...
collected 3 items / 2 deselected / 1 selected

tests/test_categories.py::test_api_call PASSED                         [100%]

=================== 1 passed, 2 deselected in 1.01s ====================
```

Only `test_api_call` matched because it's the only one with both markers.

Now, let's run tests that are marked as either `unit` OR `api`.

```bash
$ pytest -v -m "unit or api"
=========================== test session starts ============================
...
collected 3 items / 1 deselected / 2 selected

tests/test_categories.py::test_sum_numbers PASSED                      [ 50%]
tests/test_categories.py::test_api_call PASSED                         [100%]

=================== 2 passed, 1 deselected in 1.02s ====================
```

This powerful filtering is the primary reason for using markers. It allows you to slice and dice your test suite to fit any situation, from a developer's quick check to a full pre-release validation run.

## Organizing Tests by Category

## Organizing Tests by Category

Markers provide a layer of logical organization that complements the physical organization of your test files and directories. While you should still group related tests in the same file (e.g., `tests/test_user_model.py`), markers allow you to create cross-cutting categories that span your entire project.

Think of your test suite as a database. File paths are one way to query it (`pytest tests/models/`), but markers are like adding powerful, custom indexes.

### A Strategic Approach to Markers

Here are some common and effective categories for markers in a real-world project:

-   **By Speed**: `@pytest.mark.slow`, `@pytest.mark.fast`
    -   This is one of the most useful distinctions. It allows developers to run `pytest -m "not slow"` for a fast feedback loop.

-   **By Type/Layer**: `@pytest.mark.unit`, `@pytest.mark.integration`, `@pytest.mark.e2e` (end-to-end)
    -   This follows the classic testing pyramid and helps in understanding the scope of a test.

-   **By Subsystem/Feature**: `@pytest.mark.auth`, `@pytest.mark.payment`, `@pytest.mark.api_v2`
    -   This is invaluable for teams where different developers work on different parts of the application. A developer working on the payment system can run `pytest -m payment` to get focused results.

-   **By Execution Environment**: `@pytest.mark.database`, `@pytest.mark.network`, `@pytest.mark.filesystem`
    -   This helps identify tests that have external dependencies, which might not be available in all testing environments.

-   **By Purpose**: `@pytest.mark.smoke`, `@pytest.mark.regression`
    -   A `smoke` test suite is a small, critical subset of tests that can be run very quickly to see if the application is "on fire" after a deployment. `pytest -m smoke` is a perfect command for a post-deployment check.

### Example of a Well-Organized Test File

Let's see how these strategies can be combined in a single file.

```python
# tests/test_payments.py
import pytest

@pytest.mark.unit
@pytest.mark.payment
def test_credit_card_validation_format():
    # Fast, isolated check of a credit card number format
    assert True

@pytest.mark.integration
@pytest.mark.payment
@pytest.mark.database
def test_charge_user_updates_db():
    # Slower test that requires a database connection
    # to verify a user's balance is updated.
    assert True

@pytest.mark.e2e
@pytest.mark.payment
@pytest.mark.api
@pytest.mark.slow
@pytest.mark.smoke
def test_full_payment_flow():
    # Very slow test that simulates a full user journey,
    # hitting a real (or staging) payment gateway API.
    # This is also a critical "smoke" test.
    assert True
```

With this setup, you can now run your tests in many different ways, each tailored to a specific need:

-   **A developer working on payments**: `pytest -m payment`
-   **A quick pre-commit check**: `pytest -m "not slow"`
-   **CI run on every pull request**: `pytest -m "unit or integration"`
-   **A post-deployment health check**: `pytest -m smoke`
-   **Nightly full regression run**: `pytest` (runs everything)

By thoughtfully applying markers, you transform your test suite from a monolithic block of code into a flexible, queryable, and highly organized asset that can adapt to the needs of your development lifecycle.
