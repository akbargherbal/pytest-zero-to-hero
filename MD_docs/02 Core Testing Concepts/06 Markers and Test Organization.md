# Chapter 6: Markers and Test Organization

## What Are Markers?

## The Problem: Tests That Need Different Treatment

You're building a payment processing system. Your test suite includes:

- Fast unit tests that run in milliseconds
- Slow integration tests that hit a test database
- Tests that require specific environment variables
- Tests that only work on certain operating systems
- Tests for experimental features that aren't ready yet

Right now, every time you run `pytest`, all tests execute. The slow database tests make your feedback loop painful during development. You want to run only the fast tests while coding, but run everything before committing.

**The naive approach**: Create separate test files for each category. But this fragments your test organization and makes it hard to find related tests.

**What you need**: A way to tag tests with metadata and selectively run subsets based on those tags.

This is exactly what pytest markers solve.

## What Markers Are

A **marker** is metadata you attach to a test function. It's like putting a sticky note on a test that says "this test is slow" or "skip this test on Windows" or "this test requires authentication."

Markers don't change what your test does—they change how pytest treats it.

### The Simplest Marker

Let's start with a concrete example. Here's a payment processing test suite:

```python
# test_payment_processor.py

def validate_credit_card(card_number):
    """Validate credit card using Luhn algorithm."""
    if not card_number.isdigit():
        return False
    
    digits = [int(d) for d in card_number]
    checksum = 0
    
    for i, digit in enumerate(reversed(digits)):
        if i % 2 == 1:
            digit *= 2
            if digit > 9:
                digit -= 9
        checksum += digit
    
    return checksum % 10 == 0

def process_payment(amount, card_number, database_connection):
    """Process payment through payment gateway and record in database."""
    if not validate_credit_card(card_number):
        raise ValueError("Invalid credit card number")
    
    # Simulate payment gateway call (slow)
    import time
    time.sleep(2)
    
    # Record transaction in database (slow)
    database_connection.execute(
        "INSERT INTO transactions (amount, card_number) VALUES (?, ?)",
        (amount, card_number)
    )
    
    return {"status": "success", "amount": amount}

# Tests
def test_validate_credit_card_accepts_valid_number():
    """Fast unit test - runs in milliseconds."""
    valid_card = "4532015112830366"  # Valid test card
    assert validate_credit_card(valid_card) is True

def test_validate_credit_card_rejects_invalid_number():
    """Fast unit test - runs in milliseconds."""
    invalid_card = "1234567890123456"
    assert validate_credit_card(invalid_card) is False

def test_process_payment_with_valid_card(test_database):
    """Slow integration test - takes 2+ seconds."""
    result = process_payment(
        amount=100.00,
        card_number="4532015112830366",
        database_connection=test_database
    )
    assert result["status"] == "success"
    assert result["amount"] == 100.00
```

When you run this test suite, you wait over 2 seconds every time, even though the validation tests are instant. During development, you want to run only the fast tests.

**Without markers**, you'd need to:
- Put tests in separate files
- Remember which files contain which types of tests
- Manually specify file paths every time

**With markers**, you tag the slow test and filter by that tag.

### Applying Your First Marker

Here's the same test suite with a marker applied:

```python
# test_payment_processor.py
import pytest

# ... (same validate_credit_card and process_payment functions) ...

def test_validate_credit_card_accepts_valid_number():
    """Fast unit test - runs in milliseconds."""
    valid_card = "4532015112830366"
    assert validate_credit_card(valid_card) is True

def test_validate_credit_card_rejects_invalid_number():
    """Fast unit test - runs in milliseconds."""
    invalid_card = "1234567890123456"
    assert validate_credit_card(invalid_card) is False

@pytest.mark.slow
def test_process_payment_with_valid_card(test_database):
    """Slow integration test - takes 2+ seconds."""
    result = process_payment(
        amount=100.00,
        card_number="4532015112830366",
        database_connection=test_database
    )
    assert result["status"] == "success"
    assert result["amount"] == 100.00
```

**What changed**: We added `@pytest.mark.slow` above the slow test.

This is a **decorator**—a Python feature that modifies function behavior. The `@pytest.mark.slow` decorator attaches metadata to the test function without changing what the test does.

### Running Tests Selectively

Now you can control which tests run:

```bash
# Run only fast tests (skip tests marked as slow)
pytest -m "not slow"

# Run only slow tests
pytest -m "slow"

# Run all tests (default behavior)
pytest
```

**Output when running fast tests only**:

```bash
$ pytest -m "not slow" -v
======================== test session starts ========================
collected 3 items / 1 deselected / 2 selected

test_payment_processor.py::test_validate_credit_card_accepts_valid_number PASSED
test_payment_processor.py::test_validate_credit_card_rejects_invalid_number PASSED

=================== 2 passed, 1 deselected in 0.02s ===================
```

**Key observations**:

1. **"3 items / 1 deselected / 2 selected"**: Pytest found 3 tests total, deselected 1 (the slow one), and ran 2
2. **Execution time**: 0.02 seconds instead of 2+ seconds
3. **The marker didn't change the test**: It only changed whether pytest ran it

### The Anatomy of a Marker

Let's break down the syntax:

```python
@pytest.mark.slow
def test_process_payment_with_valid_card(test_database):
    # test code
```

**Components**:

- `@`: Python decorator syntax
- `pytest.mark`: The marker namespace (all pytest markers start here)
- `.slow`: The marker name (you choose this)
- Applied to: The function immediately below it

**The marker name can be anything**: `@pytest.mark.slow`, `@pytest.mark.integration`, `@pytest.mark.requires_database`, `@pytest.mark.wip` (work in progress). You define the vocabulary that makes sense for your project.

## How Markers Work Under the Hood

When pytest collects tests, it builds a list of test items. Each item has:

- The test function itself
- Metadata about the test (name, location, etc.)
- **Markers attached to it**

When you run `pytest -m "not slow"`, pytest:

1. Collects all tests
2. Checks each test's markers
3. Evaluates the expression `"not slow"` against each test's markers
4. Runs only tests where the expression is True

**The marker is stored as metadata**. You can inspect it programmatically:

```python
# conftest.py or any test file
import pytest

def test_inspect_markers():
    """Demonstrate that markers are accessible metadata."""
    # Get the current test's markers
    test_item = pytest.current_test_item  # (This is conceptual - actual access differs)
    
    # In reality, markers are accessed during test collection
    # This example shows the concept, not the exact API
```

**Why this matters**: Markers are not magic. They're structured metadata that pytest's collection and execution engine uses to make decisions. Understanding this helps you reason about more complex marker scenarios.

## Markers vs. Other Organizational Tools

You might wonder: "Why not just use separate test files or test classes?"

**Comparison**:

| Approach | Flexibility | Discoverability | Maintenance |
|----------|-------------|-----------------|-------------|
| **Separate files** | Low - tests are physically separated | Medium - must know file structure | High - moving tests requires file changes |
| **Test classes** | Medium - tests grouped but in same file | High - clear hierarchy | Medium - refactoring changes class structure |
| **Markers** | High - same test can have multiple tags | High - markers visible in test code | Low - just add/remove decorators |

**When to use markers**:

- Tests need multiple categorizations (e.g., both "slow" and "requires_auth")
- You want to run different subsets in different contexts (CI vs. local development)
- Test organization is orthogonal to code organization (e.g., integration tests for multiple modules)

**When to use files/classes instead**:

- Tests naturally group by module or feature
- You never need to run cross-cutting subsets
- Physical separation aids understanding

**Best practice**: Use both. Organize tests into logical files and classes, then use markers for cross-cutting concerns like speed, dependencies, or stability.

## What You've Learned

Markers are metadata tags you attach to tests using the `@pytest.mark.name` decorator syntax. They don't change what tests do—they change how pytest treats them.

**Core concepts**:

1. Markers are applied with `@pytest.mark.marker_name`
2. Tests are filtered with `pytest -m "expression"`
3. Markers are metadata stored on test items
4. Multiple markers can be applied to the same test
5. Markers complement (don't replace) file and class organization

**What's next**: We've used a custom marker (`slow`) without any special setup. Pytest allowed this, but issued a warning. In the next sections, we'll explore:

- Built-in markers that pytest provides
- How to properly register custom markers
- Advanced marker expressions for complex filtering
- Organizing entire test suites with marker strategies

## Built-in Markers (skip, xfail, filterwarnings)

## The Problem: Tests That Can't Always Run

Your payment processor now has tests that:

1. **Require specific platforms**: Payment gateway integration only works on Linux servers
2. **Are known to fail**: A bug in the external API causes intermittent failures
3. **Generate annoying warnings**: Deprecated library functions that you can't fix yet

Running these tests creates noise:

- Platform-specific tests fail on developer machines (Windows/Mac)
- Known failures make it hard to spot new failures
- Warning spam obscures real issues

**What you need**: Built-in markers that handle these common scenarios with well-defined behavior.

## Built-in Markers: Pytest's Standard Vocabulary

Pytest provides several markers out of the box. These markers have special meaning to pytest's execution engine—they're not just metadata, they trigger specific behaviors.

### The Three Most Important Built-in Markers

1. **`@pytest.mark.skip`**: Don't run this test at all
2. **`@pytest.mark.skipif`**: Skip this test if a condition is true
3. **`@pytest.mark.xfail`**: Run this test, but expect it to fail

Let's see each one in action with our payment processor.

## Iteration 1: Unconditional Skip with @pytest.mark.skip

### The Scenario

You're developing a new feature: cryptocurrency payment support. The implementation isn't ready, but you've written tests for it. You want these tests in version control, but they shouldn't run yet.

**First attempt**: Comment out the tests.

```python
# test_payment_processor.py

# def test_process_bitcoin_payment():
#     """Test bitcoin payment processing."""
#     result = process_payment(
#         amount=100.00,
#         payment_method="bitcoin",
#         wallet_address="1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa"
#     )
#     assert result["status"] == "success"
```

**Problems with this approach**:

- Test is invisible to pytest (won't show in `--collect-only`)
- Easy to forget about commented tests
- No documentation of why it's disabled
- Can't track skipped tests in reports

### The Solution: @pytest.mark.skip

Mark the test as skipped with a reason:

```python
# test_payment_processor.py
import pytest

@pytest.mark.skip(reason="Bitcoin payment not implemented yet")
def test_process_bitcoin_payment():
    """Test bitcoin payment processing."""
    result = process_payment(
        amount=100.00,
        payment_method="bitcoin",
        wallet_address="1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa"
    )
    assert result["status"] == "success"
```

**Run the test suite**:

```bash
$ pytest -v
======================== test session starts ========================
collected 4 items

test_payment_processor.py::test_validate_credit_card_accepts_valid_number PASSED
test_payment_processor.py::test_validate_credit_card_rejects_invalid_number PASSED
test_payment_processor.py::test_process_payment_with_valid_card PASSED
test_payment_processor.py::test_process_bitcoin_payment SKIPPED (Bitcoin payment not implemented yet)

=================== 3 passed, 1 skipped in 0.03s ===================
```

**What happened**:

1. Pytest collected the test (it's not invisible)
2. Pytest didn't run the test code
3. The test shows as `SKIPPED` with your reason
4. The summary shows `1 skipped`

**Key insight**: The test function body never executes. Pytest sees the `@pytest.mark.skip` decorator and immediately marks the test as skipped during collection.

### When to Use Unconditional Skip

Use `@pytest.mark.skip` when:

- Feature not implemented yet (like our bitcoin example)
- Test is temporarily broken and you need to commit
- Test requires manual setup that's not automated

**Don't use it for**:

- Platform-specific tests (use `skipif` instead)
- Tests that should fail (use `xfail` instead)
- Tests you'll never fix (delete them instead)

## Iteration 2: Conditional Skip with @pytest.mark.skipif

### The Scenario

Your payment gateway integration requires Linux because it uses Linux-specific system calls. On Windows and Mac, the test should skip automatically.

**First attempt**: Manual platform check inside the test.

```python
# test_payment_processor.py
import sys
import pytest

def test_payment_gateway_integration(test_database):
    """Test integration with payment gateway (Linux only)."""
    if sys.platform != "linux":
        pytest.skip("Payment gateway only available on Linux")
    
    # Test code here
    result = process_payment_via_gateway(
        amount=100.00,
        card_number="4532015112830366"
    )
    assert result["gateway_response"] == "approved"
```

**Problems**:

- Test function executes partially (runs the if statement)
- Skip logic is inside the test (not visible in test list)
- Every platform-specific test needs this boilerplate

### The Solution: @pytest.mark.skipif

Move the condition to the decorator:

```python
# test_payment_processor.py
import sys
import pytest

@pytest.mark.skipif(
    sys.platform != "linux",
    reason="Payment gateway only available on Linux"
)
def test_payment_gateway_integration(test_database):
    """Test integration with payment gateway (Linux only)."""
    result = process_payment_via_gateway(
        amount=100.00,
        card_number="4532015112830366"
    )
    assert result["gateway_response"] == "approved"
```

**Run on macOS**:

```bash
$ pytest -v test_payment_processor.py::test_payment_gateway_integration
======================== test session starts ========================
collected 1 item

test_payment_processor.py::test_payment_gateway_integration SKIPPED (Payment gateway only available on Linux)

=================== 1 skipped in 0.01s ===================
```

**Run on Linux**:

```bash
$ pytest -v test_payment_processor.py::test_payment_gateway_integration
======================== test session starts ========================
collected 1 item

test_payment_processor.py::test_payment_gateway_integration PASSED

=================== 1 passed in 2.34s ===================
```

**What changed**:

1. **On macOS**: Test skipped during collection (function never runs)
2. **On Linux**: Test runs normally
3. **Visibility**: The skip condition is visible in the test signature

### Common skipif Conditions

Here are real-world examples:

```python
import sys
import pytest

# Platform-specific
@pytest.mark.skipif(sys.platform == "win32", reason="Unix-only test")
def test_unix_file_permissions():
    pass

@pytest.mark.skipif(sys.platform != "darwin", reason="macOS-only test")
def test_macos_keychain_integration():
    pass

# Python version-specific
@pytest.mark.skipif(
    sys.version_info < (3, 10),
    reason="Requires Python 3.10+ match statement"
)
def test_pattern_matching():
    pass

# Dependency-specific
@pytest.mark.skipif(
    not pytest.importorskip("requests"),
    reason="Requires requests library"
)
def test_http_client():
    pass

# Environment-specific
@pytest.mark.skipif(
    "CI" not in os.environ,
    reason="Only runs in CI environment"
)
def test_deployment_pipeline():
    pass
```

### Reusing Skip Conditions

When multiple tests share the same skip condition, define it once:

```python
# test_payment_processor.py
import sys
import pytest

# Define the condition once
requires_linux = pytest.mark.skipif(
    sys.platform != "linux",
    reason="Requires Linux"
)

# Apply to multiple tests
@requires_linux
def test_payment_gateway_integration(test_database):
    pass

@requires_linux
def test_payment_gateway_refund(test_database):
    pass

@requires_linux
def test_payment_gateway_batch_processing(test_database):
    pass
```

**Benefits**:

- Single source of truth for the condition
- Easy to update if logic changes
- Self-documenting test requirements

## Iteration 3: Expected Failures with @pytest.mark.xfail

### The Scenario

The payment gateway API has a known bug: refunds over $10,000 fail with a 500 error. You've reported it to the vendor, but it's not fixed yet. You want to:

1. Document the bug with a test
2. Track when it's fixed
3. Not have this failure hide other failures

**First attempt**: Skip the test.

```python
@pytest.mark.skip(reason="Gateway bug: refunds over $10k fail")
def test_large_refund():
    """Test refund of large amount."""
    result = process_refund(amount=15000.00)
    assert result["status"] == "success"
```

**Problem**: When the bug is fixed, you won't know. The test will stay skipped forever.

### The Solution: @pytest.mark.xfail

Mark the test as expected to fail:

```python
# test_payment_processor.py
import pytest

@pytest.mark.xfail(reason="Gateway bug: refunds over $10k fail with 500 error")
def test_large_refund():
    """Test refund of large amount."""
    result = process_refund(amount=15000.00)
    assert result["status"] == "success"
```

**Run the test while the bug exists**:

```bash
$ pytest -v test_payment_processor.py::test_large_refund
======================== test session starts ========================
collected 1 item

test_payment_processor.py::test_large_refund XFAIL (Gateway bug: refunds over $10k fail with 500 error)

=================== 1 xfailed in 0.45s ===================
```

**What happened**:

1. Pytest **ran the test** (unlike `skip`)
2. The test **failed** as expected
3. Pytest marked it as `XFAIL` (expected failure)
4. The test suite still passes overall

**Now the bug gets fixed**. Run the test again:

```bash
$ pytest -v test_payment_processor.py::test_large_refund
======================== test session starts ========================
collected 1 item

test_payment_processor.py::test_large_refund XPASS (Gateway bug: refunds over $10k fail with 500 error)

=================== 1 xpassed in 0.45s ===================
```

**Critical difference**: The test shows `XPASS` (unexpectedly passed). This alerts you that:

1. The bug is fixed
2. You can remove the `xfail` marker
3. The test should now be a regular passing test

### xfail vs. skip: When to Use Each

| Scenario | Use | Reason |
|----------|-----|--------|
| Feature not implemented | `skip` | No point running the test |
| Known bug in your code | `xfail` | Track when it's fixed |
| Known bug in external dependency | `xfail` | Track when vendor fixes it |
| Platform not supported | `skipif` | Test can't run here |
| Flaky test being investigated | `xfail` | Don't block CI while debugging |

### xfail with Conditions

You can combine `xfail` with conditions:

```python
@pytest.mark.xfail(
    sys.platform == "win32",
    reason="Known issue on Windows - investigating"
)
def test_file_locking():
    """Test file locking mechanism."""
    # This test passes on Linux/Mac but fails on Windows
    pass
```

**Behavior**:

- On Windows: Runs and expects failure (XFAIL if fails, XPASS if passes)
- On Linux/Mac: Runs normally (PASS or FAIL)

### Strict xfail: Fail if Test Passes

Sometimes you want to be notified immediately when an xfail test passes:

```python
@pytest.mark.xfail(
    reason="Gateway bug: refunds over $10k fail",
    strict=True
)
def test_large_refund():
    """Test refund of large amount."""
    result = process_refund(amount=15000.00)
    assert result["status"] == "success"
```

**With `strict=True`**:

- If test fails: XFAIL (expected, test suite passes)
- If test passes: **FAILED** (unexpected, test suite fails)

**Use strict mode when**: You want CI to fail if an expected failure suddenly passes, forcing you to update the test immediately.

## Iteration 4: Filtering Warnings with @pytest.mark.filterwarnings

### The Scenario

Your payment processor uses a library that's deprecated but still works. Every test run shows:

```bash
$ pytest
======================== test session starts ========================
collected 10 items

test_payment_processor.py::test_validate_credit_card PASSED
test_payment_processor.py::test_process_payment PASSED
  /path/to/payment_lib.py:45: DeprecationWarning: process_payment_v1 is deprecated, use process_payment_v2
    warnings.warn("process_payment_v1 is deprecated, use process_payment_v2", DeprecationWarning)

=================== 2 passed, 1 warning in 0.05s ===================
```

**Problem**: The warning spam makes it hard to see real issues. You can't fix it yet (migration planned for next quarter), but you want clean test output.

### The Solution: @pytest.mark.filterwarnings

Suppress specific warnings for specific tests:

```python
# test_payment_processor.py
import pytest

@pytest.mark.filterwarnings("ignore::DeprecationWarning")
def test_process_payment_legacy_api():
    """Test payment processing using legacy API."""
    # This test uses the deprecated function
    result = process_payment_v1(amount=100.00)
    assert result["status"] == "success"

def test_process_payment_new_api():
    """Test payment processing using new API."""
    # This test uses the new function (no warning)
    result = process_payment_v2(amount=100.00)
    assert result["status"] == "success"
```

**Run the tests**:

```bash
$ pytest -v
======================== test session starts ========================
collected 2 items

test_payment_processor.py::test_process_payment_legacy_api PASSED
test_payment_processor.py::test_process_payment_new_api PASSED

=================== 2 passed in 0.03s ===================
```

**What happened**: The deprecation warning from `test_process_payment_legacy_api` was suppressed. Other warnings still appear.

### Warning Filter Syntax

The filter string follows Python's warning filter format:

```python
# Ignore all deprecation warnings
@pytest.mark.filterwarnings("ignore::DeprecationWarning")

# Ignore specific warning message
@pytest.mark.filterwarnings("ignore:process_payment_v1 is deprecated")

# Turn warnings into errors (strict mode)
@pytest.mark.filterwarnings("error::DeprecationWarning")

# Ignore warnings from specific module
@pytest.mark.filterwarnings("ignore::DeprecationWarning:payment_lib")
```

### When to Use filterwarnings

**Good uses**:

- Suppress known warnings from third-party libraries you can't fix
- Test code that intentionally triggers warnings
- Temporarily silence warnings during migration

**Bad uses**:

- Hiding warnings in your own code (fix them instead)
- Suppressing all warnings globally (you'll miss real issues)
- Using it as a permanent solution (warnings indicate technical debt)

## Diagnostic Analysis: Understanding Marker Behavior

Let's intentionally create scenarios to understand how markers work:

### Scenario 1: What Happens When xfail Test Passes?

```python
# test_marker_behavior.py
import pytest

@pytest.mark.xfail(reason="Expected to fail")
def test_xfail_that_actually_passes():
    """This test is marked xfail but will pass."""
    assert 1 + 1 == 2  # This will pass
```

**Run it**:

```bash
$ pytest -v test_marker_behavior.py::test_xfail_that_actually_passes
======================== test session starts ========================
collected 1 item

test_marker_behavior.py::test_xfail_that_actually_passes XPASS (Expected to fail)

=================== 1 xpassed in 0.01s ===================
```

**Analysis**:

1. **Status**: `XPASS` (unexpectedly passed)
2. **Test suite result**: Still passes (xpass doesn't fail the suite by default)
3. **What this tells us**: xfail tests always run; the marker only changes how pytest interprets the result

### Scenario 2: Combining Multiple Markers

```python
# test_marker_behavior.py
import sys
import pytest

@pytest.mark.slow
@pytest.mark.skipif(sys.platform == "win32", reason="Unix only")
def test_with_multiple_markers():
    """Test with both slow and skipif markers."""
    assert True
```

**Run on Windows**:

```bash
$ pytest -v test_marker_behavior.py::test_with_multiple_markers
======================== test session starts ========================
collected 1 item

test_marker_behavior.py::test_with_multiple_markers SKIPPED (Unix only)

=================== 1 skipped in 0.01s ===================
```

**Analysis**:

1. **Skip takes precedence**: Test never runs, so the `slow` marker is irrelevant
2. **Marker order doesn't matter**: `skipif` is evaluated during collection, before execution
3. **What this tells us**: Markers are evaluated in a specific order (skip/skipif first, then xfail, then custom markers)

### Scenario 3: Skip Inside Test vs. Skip Marker

```python
# test_marker_behavior.py
import pytest

def test_skip_inside_function():
    """Skip called inside the test function."""
    pytest.skip("Skipping for demonstration")
    assert False  # This line never executes

@pytest.mark.skip(reason="Skipping for demonstration")
def test_skip_with_marker():
    """Skip using marker."""
    assert False  # This line never executes
```

**Run both**:

```bash
$ pytest -v test_marker_behavior.py -k "skip"
======================== test session starts ========================
collected 2 items

test_marker_behavior.py::test_skip_inside_function SKIPPED (Skipping for demonstration)
test_marker_behavior.py::test_skip_with_marker SKIPPED (Skipping for demonstration)

=================== 2 skipped in 0.02s ===================
```

**Analysis**:

1. **Both show as SKIPPED**: Same end result
2. **Timing difference**: Marker skip happens during collection (0ms), function skip happens during execution (after setup)
3. **When to use each**:
   - Marker: Condition known before test runs (platform, Python version)
   - Function: Condition determined during test execution (missing file, API unavailable)

## Built-in Markers Summary

| Marker | Purpose | Test Runs? | Use When |
|--------|---------|------------|----------|
| `@pytest.mark.skip` | Don't run this test | No | Feature not ready, test broken |
| `@pytest.mark.skipif(condition)` | Skip if condition true | No | Platform/version specific |
| `@pytest.mark.xfail` | Expect this test to fail | Yes | Known bug, flaky test |
| `@pytest.mark.filterwarnings` | Control warning display | Yes | Suppress known warnings |

### Decision Framework: Which Marker to Use?

**Question 1**: Should the test run at all?

- **No** → Use `skip` or `skipif`
- **Yes** → Continue to Question 2

**Question 2**: Do you expect it to pass?

- **Yes** → No marker needed (or use custom markers for organization)
- **No** → Use `xfail`

**Question 3**: Does it generate warnings?

- **Yes, and I can't fix them** → Add `filterwarnings`
- **Yes, and I should fix them** → Don't suppress, fix the code

## What You've Learned

Built-in markers provide standard solutions for common testing scenarios:

1. **`skip`/`skipif`**: Prevent tests from running based on conditions
2. **`xfail`**: Run tests that are expected to fail, track when they pass
3. **`filterwarnings`**: Control warning output for specific tests

**Key insights**:

- Markers are evaluated in order: skip → xfail → execution
- `xfail` tests always run (unlike `skip`)
- Markers can be combined, but skip takes precedence
- Built-in markers have special behavior (not just metadata)

**What's next**: Built-in markers cover common cases, but real projects need custom markers for domain-specific organization. In the next section, we'll create custom markers for our payment processor test suite.

## Creating Custom Markers

## The Problem: Project-Specific Test Categories

Your payment processor test suite has grown. You now have:

- **Unit tests**: Fast, isolated, no external dependencies
- **Integration tests**: Hit test database, slower
- **End-to-end tests**: Full payment flow, very slow
- **Security tests**: Test fraud detection, PCI compliance
- **Performance tests**: Benchmark payment processing speed

You want to:

- Run only unit tests during development (fast feedback)
- Run integration tests before committing
- Run security tests before deployment
- Run performance tests nightly

Built-in markers (`skip`, `xfail`) don't fit these categories. You need **custom markers** that reflect your project's structure.

## Iteration 1: Creating Your First Custom Marker

### The Naive Approach

Let's try using a custom marker without any setup:

```python
# test_payment_processor.py
import pytest

@pytest.mark.unit
def test_validate_credit_card():
    """Fast unit test."""
    assert validate_credit_card("4532015112830366") is True

@pytest.mark.integration
def test_process_payment_with_database(test_database):
    """Integration test with database."""
    result = process_payment(100.00, "4532015112830366", test_database)
    assert result["status"] == "success"
```

**Run the tests**:

```bash
$ pytest -v
======================== test session starts ========================
collected 2 items

test_payment_processor.py::test_validate_credit_card PASSED
test_payment_processor.py::test_validate_credit_card PASSED

=================== 2 passed, 1 warning in 0.03s ===================

warnings summary
test_payment_processor.py:4
  /path/to/test_payment_processor.py:4: PytestUnknownMarkWarning: Unknown pytest.mark.unit - is this a typo?  You can register custom marks to avoid this warning - for details, see https://docs.pytest.org/en/stable/how-to/mark.html
    @pytest.mark.unit

test_payment_processor.py:9
  /path/to/test_payment_processor.py:9: PytestUnknownMarkWarning: Unknown pytest.mark.integration - is this a typo?  You can register custom marks to avoid this warning - for details, see https://docs.pytest.org/en/stable/how-to/mark.html
    @pytest.mark.integration
```

**What happened**:

1. ✅ Tests ran successfully
2. ✅ Markers were applied (you can filter with `-m unit`)
3. ⚠️ Pytest issued warnings about unknown markers

**Why the warning?**: Pytest doesn't know if `unit` and `integration` are:

- Intentional custom markers
- Typos (e.g., `@pytest.mark.integratoin`)
- Mistakes (e.g., `@pytest.mark.slow` when you meant the built-in)

### The Solution: Register Custom Markers

Tell pytest about your custom markers in a configuration file. Create `pytest.ini`:

```ini
# pytest.ini
[pytest]
markers =
    unit: Fast unit tests with no external dependencies
    integration: Integration tests that use test database
```

**Run the tests again**:

```bash
$ pytest -v
======================== test session starts ========================
collected 2 items

test_payment_processor.py::test_validate_credit_card PASSED
test_payment_processor.py::test_process_payment_with_database PASSED

=================== 2 passed in 0.03s ===================
```

**What changed**:

1. ✅ No warnings
2. ✅ Markers are now "official" parts of your test suite
3. ✅ Typos will be caught (e.g., `@pytest.mark.integratoin` will still warn)

### Viewing Registered Markers

See all available markers:

```bash
$ pytest --markers
@pytest.mark.unit: Fast unit tests with no external dependencies

@pytest.mark.integration: Integration tests that use test database

@pytest.mark.skip(reason=None): skip the given test function with an optional reason...

@pytest.mark.skipif(condition, ..., *, reason=...): skip the given test function if any of the conditions evaluate to True...

@pytest.mark.xfail(condition, ..., *, reason=..., run=True, raises=None, strict=xfail_strict): mark test as expected to fail...

@pytest.mark.filterwarnings(warning): add a warning filter to the given test...

@pytest.mark.parametrize(argnames, argvalues): call a test function multiple times passing in different arguments...

[... more built-in markers ...]
```

**Key observation**: Your custom markers appear alongside built-in markers, with your descriptions visible.

## Iteration 2: Building a Complete Marker Taxonomy

### The Scenario

Your payment processor now has a comprehensive test suite. You need markers for:

1. **Test type**: unit, integration, e2e
2. **Test speed**: fast, slow
3. **Test category**: security, performance, smoke
4. **Requirements**: requires_database, requires_network, requires_auth

Let's build a complete marker system:

```ini
# pytest.ini
[pytest]
markers =
    # Test types
    unit: Fast unit tests with no external dependencies
    integration: Integration tests that use test database or external services
    e2e: End-to-end tests that test complete user workflows
    
    # Speed categories
    fast: Tests that complete in under 100ms
    slow: Tests that take more than 1 second
    
    # Functional categories
    security: Security-related tests (fraud detection, PCI compliance)
    performance: Performance and load testing
    smoke: Critical path tests that should always pass
    
    # Requirements
    requires_database: Test requires database connection
    requires_network: Test requires network access
    requires_auth: Test requires authentication setup
```

**Now apply these markers to your test suite**:

```python
# test_payment_processor.py
import pytest

# ============================================================================
# Unit Tests - Fast, No Dependencies
# ============================================================================

@pytest.mark.unit
@pytest.mark.fast
def test_validate_credit_card_accepts_valid_number():
    """Validate credit card using Luhn algorithm."""
    valid_card = "4532015112830366"
    assert validate_credit_card(valid_card) is True

@pytest.mark.unit
@pytest.mark.fast
def test_validate_credit_card_rejects_invalid_number():
    """Reject invalid credit card number."""
    invalid_card = "1234567890123456"
    assert validate_credit_card(invalid_card) is False

@pytest.mark.unit
@pytest.mark.fast
def test_calculate_transaction_fee():
    """Calculate correct transaction fee."""
    fee = calculate_transaction_fee(amount=100.00, card_type="visa")
    assert fee == 2.90  # 2.9% for Visa

# ============================================================================
# Integration Tests - Database Required
# ============================================================================

@pytest.mark.integration
@pytest.mark.slow
@pytest.mark.requires_database
def test_process_payment_records_transaction(test_database):
    """Process payment and verify database record."""
    result = process_payment(
        amount=100.00,
        card_number="4532015112830366",
        database_connection=test_database
    )
    
    # Verify transaction recorded
    cursor = test_database.execute(
        "SELECT * FROM transactions WHERE amount = ?", (100.00,)
    )
    transaction = cursor.fetchone()
    assert transaction is not None
    assert transaction["status"] == "success"

@pytest.mark.integration
@pytest.mark.slow
@pytest.mark.requires_database
def test_refund_updates_transaction_status(test_database):
    """Process refund and verify status update."""
    # First create a transaction
    process_payment(100.00, "4532015112830366", test_database)
    
    # Then refund it
    refund_payment(transaction_id=1, database_connection=test_database)
    
    # Verify status updated
    cursor = test_database.execute(
        "SELECT status FROM transactions WHERE id = ?", (1,)
    )
    status = cursor.fetchone()["status"]
    assert status == "refunded"

# ============================================================================
# End-to-End Tests - Full Workflow
# ============================================================================

@pytest.mark.e2e
@pytest.mark.slow
@pytest.mark.requires_database
@pytest.mark.requires_network
def test_complete_payment_workflow(test_database, mock_payment_gateway):
    """Test complete payment workflow from cart to confirmation."""
    # 1. Create shopping cart
    cart = create_cart(items=[
        {"product_id": 1, "quantity": 2, "price": 29.99},
        {"product_id": 2, "quantity": 1, "price": 49.99}
    ])
    
    # 2. Process payment
    payment_result = process_payment(
        amount=cart.total,
        card_number="4532015112830366",
        database_connection=test_database
    )
    
    # 3. Verify payment gateway called
    assert mock_payment_gateway.was_called()
    
    # 4. Verify order created
    order = get_order(payment_result["order_id"])
    assert order.status == "confirmed"
    assert order.total == cart.total

# ============================================================================
# Security Tests
# ============================================================================

@pytest.mark.security
@pytest.mark.unit
@pytest.mark.fast
def test_credit_card_number_is_masked_in_logs():
    """Ensure credit card numbers are masked in log output."""
    card_number = "4532015112830366"
    log_entry = create_log_entry(card_number)
    
    # Should show only last 4 digits
    assert "4532015112830366" not in log_entry
    assert "****0366" in log_entry

@pytest.mark.security
@pytest.mark.integration
@pytest.mark.slow
def test_fraud_detection_blocks_suspicious_transaction(test_database):
    """Test fraud detection system blocks suspicious patterns."""
    # Attempt multiple rapid transactions from same card
    card_number = "4532015112830366"
    
    for i in range(10):
        result = process_payment(
            amount=1000.00,
            card_number=card_number,
            database_connection=test_database
        )
    
    # 10th transaction should be blocked
    assert result["status"] == "blocked"
    assert result["reason"] == "fraud_detection"

# ============================================================================
# Performance Tests
# ============================================================================

@pytest.mark.performance
@pytest.mark.slow
def test_payment_processing_performance(benchmark):
    """Benchmark payment processing speed."""
    result = benchmark(
        process_payment,
        amount=100.00,
        card_number="4532015112830366",
        database_connection=None  # Use mock
    )
    
    # Should process in under 100ms
    assert benchmark.stats.mean < 0.1

# ============================================================================
# Smoke Tests - Critical Path
# ============================================================================

@pytest.mark.smoke
@pytest.mark.fast
def test_payment_processor_is_importable():
    """Verify payment processor module can be imported."""
    from payment_processor import process_payment
    assert callable(process_payment)

@pytest.mark.smoke
@pytest.mark.integration
@pytest.mark.requires_database
def test_can_connect_to_database(test_database):
    """Verify database connection works."""
    cursor = test_database.execute("SELECT 1")
    assert cursor.fetchone()[0] == 1
```

**What we've built**:

1. **Clear categorization**: Every test has meaningful markers
2. **Multiple dimensions**: Tests can be unit + fast + security
3. **Flexible filtering**: Can run tests by type, speed, category, or requirements
4. **Self-documenting**: Markers explain what each test needs

## Iteration 3: Using Markers for Selective Test Execution

Now that tests are marked, let's use markers to run specific subsets:

### Development Workflow: Fast Feedback

```bash
# Run only fast unit tests during development
$ pytest -m "unit and fast" -v
======================== test session starts ========================
collected 25 items / 20 deselected / 5 selected

test_payment_processor.py::test_validate_credit_card_accepts_valid_number PASSED
test_payment_processor.py::test_validate_credit_card_rejects_invalid_number PASSED
test_payment_processor.py::test_calculate_transaction_fee PASSED
test_payment_processor.py::test_credit_card_number_is_masked_in_logs PASSED
test_payment_processor.py::test_payment_processor_is_importable PASSED

=================== 5 passed, 20 deselected in 0.05s ===================
```

**Result**: 5 tests in 0.05 seconds. Perfect for TDD cycle.

### Pre-Commit: Integration Tests

```bash
# Run unit and integration tests before committing
$ pytest -m "unit or integration" -v
======================== test session starts ========================
collected 25 items / 8 deselected / 17 selected

test_payment_processor.py::test_validate_credit_card_accepts_valid_number PASSED
test_payment_processor.py::test_validate_credit_card_rejects_invalid_number PASSED
test_payment_processor.py::test_calculate_transaction_fee PASSED
test_payment_processor.py::test_process_payment_records_transaction PASSED
test_payment_processor.py::test_refund_updates_transaction_status PASSED
test_payment_processor.py::test_credit_card_number_is_masked_in_logs PASSED
test_payment_processor.py::test_fraud_detection_blocks_suspicious_transaction PASSED
test_payment_processor.py::test_payment_processor_is_importable PASSED
test_payment_processor.py::test_can_connect_to_database PASSED
[... more tests ...]

=================== 17 passed, 8 deselected in 2.34s ===================
```

**Result**: 17 tests in 2.34 seconds. Reasonable for pre-commit hook.

### CI Pipeline: Security and Smoke Tests

```bash
# Run security and smoke tests in CI
$ pytest -m "security or smoke" -v
======================== test session starts ========================
collected 25 items / 20 deselected / 5 selected

test_payment_processor.py::test_credit_card_number_is_masked_in_logs PASSED
test_payment_processor.py::test_fraud_detection_blocks_suspicious_transaction PASSED
test_payment_processor.py::test_payment_processor_is_importable PASSED
test_payment_processor.py::test_can_connect_to_database PASSED

=================== 4 passed, 20 deselected in 1.23s ===================
```

### Nightly Build: Everything

```bash
# Run all tests including slow performance tests
$ pytest -v
======================== test session starts ========================
collected 25 items

[... all tests run ...]

=================== 25 passed in 15.67s ===================
```

### Complex Marker Expressions

Pytest supports boolean logic in marker expressions:

```bash
# Run integration tests that don't require network
$ pytest -m "integration and not requires_network"

# Run fast tests OR smoke tests
$ pytest -m "fast or smoke"

# Run security tests that are also unit tests
$ pytest -m "security and unit"

# Run everything except slow tests
$ pytest -m "not slow"

# Run tests that require database but not network
$ pytest -m "requires_database and not requires_network"
```

**Operator precedence**:

1. `not` (highest)
2. `and`
3. `or` (lowest)

Use parentheses for clarity: `pytest -m "(unit or integration) and not slow"`

## Iteration 4: Markers with Arguments

### The Scenario

Some tests require specific Python versions or external service versions. You want to document these requirements in the marker itself.

**Custom markers can accept arguments**:

```ini
# pytest.ini
[pytest]
markers =
    # ... previous markers ...
    requires_python(version): Test requires specific Python version
    requires_service(name, version): Test requires external service
```

```python
# test_payment_processor.py
import sys
import pytest

@pytest.mark.requires_python("3.10")
@pytest.mark.skipif(sys.version_info < (3, 10), reason="Requires Python 3.10+")
def test_pattern_matching_in_payment_routing():
    """Test payment routing using Python 3.10 match statement."""
    payment_type = "credit_card"
    
    match payment_type:
        case "credit_card":
            processor = CreditCardProcessor()
        case "paypal":
            processor = PayPalProcessor()
        case _:
            processor = DefaultProcessor()
    
    assert isinstance(processor, CreditCardProcessor)

@pytest.mark.requires_service("payment_gateway", "v2.0")
def test_payment_gateway_v2_features():
    """Test features specific to payment gateway v2.0."""
    # Test code here
    pass
```

**Why use markers with arguments?**

1. **Documentation**: The marker shows requirements clearly
2. **Filtering**: Can filter by specific versions if needed
3. **Reporting**: Test reports show which versions are required
4. **Automation**: CI can parse markers to set up correct environment

**Note**: The marker itself doesn't enforce the requirement—you still need `skipif` for that. The marker is documentation and metadata.

## Diagnostic Analysis: Common Marker Mistakes

### Mistake 1: Forgetting to Register Markers

**Symptom**: Warning about unknown marker.

```bash
PytestUnknownMarkWarning: Unknown pytest.mark.mymarker - is this a typo?
```

**Root cause**: Marker used but not registered in `pytest.ini`.

**Solution**: Add to `pytest.ini`:

```ini
[pytest]
markers =
    mymarker: Description of what this marker means
```

### Mistake 2: Typo in Marker Name

**Symptom**: Tests don't run when filtering by marker.

```bash
$ pytest -m "integratoin"  # Typo: should be "integration"
======================== test session starts ========================
collected 25 items / 25 deselected / 0 selected

=================== 25 deselected in 0.02s ===================
```

**Root cause**: Marker name in filter doesn't match marker name in code.

**Solution**: 

1. Check marker spelling in test code
2. Use `pytest --markers` to see registered markers
3. Consider using constants for marker names:

```python
# conftest.py
MARKER_UNIT = "unit"
MARKER_INTEGRATION = "integration"

# test_payment_processor.py
import pytest
from conftest import MARKER_UNIT

@pytest.mark.__getattr__(MARKER_UNIT)  # Not recommended, just showing concept
def test_something():
    pass
```

**Better approach**: Use IDE autocomplete and rely on the warning system to catch typos.

### Mistake 3: Over-Marking Tests

**Symptom**: Tests have 5+ markers, making them hard to understand.

```python
@pytest.mark.unit
@pytest.mark.fast
@pytest.mark.security
@pytest.mark.smoke
@pytest.mark.critical
@pytest.mark.regression
def test_something():
    pass
```

**Root cause**: Trying to categorize tests in too many dimensions.

**Solution**: Choose 2-3 primary dimensions:

1. **Test type** (unit/integration/e2e)
2. **Speed** (fast/slow)
3. **Category** (security/performance/smoke)

**Guideline**: If a test needs more than 3 markers, your marker taxonomy is too complex.

### Mistake 4: Inconsistent Marker Usage

**Symptom**: Some integration tests marked `integration`, others not.

```python
# Some tests marked
@pytest.mark.integration
def test_payment_with_database(test_database):
    pass

# Others not marked
def test_refund_with_database(test_database):  # Missing marker!
    pass
```

**Root cause**: No enforcement of marker policy.

**Solution**: Create a pytest hook to enforce markers:

```python
# conftest.py
import pytest

def pytest_collection_modifyitems(config, items):
    """Ensure all tests have required markers."""
    for item in items:
        # Get all markers on this test
        markers = [mark.name for mark in item.iter_markers()]
        
        # Check if test has at least one type marker
        type_markers = {"unit", "integration", "e2e"}
        if not any(m in type_markers for m in markers):
            # Add default marker or raise error
            item.add_marker(pytest.mark.unit)  # Default to unit
            
            # Or raise error to force explicit marking:
            # pytest.fail(f"Test {item.nodeid} missing type marker (unit/integration/e2e)")
```

**This hook runs during test collection and can**:

- Add default markers
- Validate marker presence
- Enforce marker policies

## When to Apply Custom Markers

### Use Custom Markers When:

1. **Multiple test categories exist**: unit, integration, e2e
2. **Different execution contexts**: local dev, CI, nightly builds
3. **Resource requirements vary**: database, network, authentication
4. **Speed matters**: fast feedback loop vs. comprehensive testing
5. **Compliance needs**: security, performance, smoke tests

### Don't Use Custom Markers When:

1. **Only one test type**: Just use files/directories
2. **Never run subsets**: Always run all tests
3. **Categories are obvious**: File structure already clear
4. **Over-engineering**: 5 tests don't need 10 markers

### Decision Framework: Files vs. Markers

| Scenario | Use | Reason |
|----------|-----|--------|
| Tests naturally group by module | Files/directories | Physical organization matches logical |
| Tests span multiple modules | Markers | Cross-cutting concerns |
| Never run partial suite | Files | No need for filtering |
| Frequently run subsets | Markers | Flexible filtering |
| Team < 3 people | Files | Simpler |
| Team > 10 people | Markers | Standardized categories |

## What You've Learned

Custom markers let you create project-specific test categories:

1. **Register markers** in `pytest.ini` to avoid warnings
2. **Apply multiple markers** to tests for multi-dimensional categorization
3. **Filter tests** using boolean expressions: `and`, `or`, `not`
4. **Markers with arguments** document requirements
5. **Enforce marker policies** with pytest hooks

**Key insights**:

- Markers are metadata, not behavior (except built-in markers)
- Good marker taxonomy has 2-3 dimensions
- Markers enable flexible test execution strategies
- Registration prevents typos and documents intent

**What's next**: We've created markers and applied them to tests. But where do you register them? How do you share marker definitions across projects? The next section covers marker registration and configuration in depth.

## Registering Markers in Configuration

## The Problem: Marker Configuration Chaos

Your payment processor project has grown. You now have:

- Multiple test files using the same markers
- New team members who don't know which markers exist
- Typos in marker names causing silent failures
- No documentation of what each marker means

**Current state**: Markers work, but there's no central source of truth.

**What you need**: A configuration system that:

1. Defines all available markers in one place
2. Documents what each marker means
3. Catches typos automatically
4. Works across different configuration formats

This is what marker registration solves.

## Iteration 1: Basic Registration in pytest.ini

### The Scenario

You have three test files, all using the same markers:

```python
# test_payment_processor.py
import pytest

@pytest.mark.unit
def test_validate_credit_card():
    pass

@pytest.mark.integration
def test_process_payment(test_database):
    pass

# test_fraud_detection.py
import pytest

@pytest.mark.unit
def test_fraud_score_calculation():
    pass

@pytest.mark.security
def test_fraud_detection_blocks_suspicious():
    pass

# test_reporting.py
import pytest

@pytest.mark.integration
def test_generate_transaction_report(test_database):
    pass

@pytest.mark.slow
def test_generate_annual_report(test_database):
    pass
```

**Without registration**, running tests produces warnings:

```bash
$ pytest
======================== test session starts ========================
collected 6 items

test_payment_processor.py ..                                   [ 33%]
test_fraud_detection.py ..                                     [ 66%]
test_reporting.py ..                                           [100%]

=================== 6 passed, 4 warnings in 0.15s ===================

warnings summary
test_payment_processor.py:3
  PytestUnknownMarkWarning: Unknown pytest.mark.unit
test_payment_processor.py:7
  PytestUnknownMarkWarning: Unknown pytest.mark.integration
test_fraud_detection.py:7
  PytestUnknownMarkWarning: Unknown pytest.mark.security
test_reporting.py:7
  PytestUnknownMarkWarning: Unknown pytest.mark.slow
```

### The Solution: Create pytest.ini

Create a `pytest.ini` file in your project root:

```ini
# pytest.ini
[pytest]
markers =
    unit: Fast unit tests with no external dependencies
    integration: Integration tests that use test database
    security: Security-related tests
    slow: Tests that take more than 1 second
```

**Run tests again**:

```bash
$ pytest
======================== test session starts ========================
collected 6 items

test_payment_processor.py ..                                   [ 33%]
test_fraud_detection.py ..                                     [ 66%]
test_reporting.py ..                                           [100%]

=================== 6 passed in 0.15s ===================
```

**What changed**:

1. ✅ No warnings
2. ✅ All markers recognized
3. ✅ Single source of truth for marker definitions

### Viewing Registered Markers

Check what markers are available:

```bash
$ pytest --markers
@pytest.mark.unit: Fast unit tests with no external dependencies

@pytest.mark.integration: Integration tests that use test database

@pytest.mark.security: Security-related tests

@pytest.mark.slow: Tests that take more than 1 second

@pytest.mark.skip(reason=None): skip the given test function...
[... built-in markers ...]
```

**Key observation**: Your custom markers appear first, with descriptions, followed by built-in markers.

## Iteration 2: Alternative Configuration Formats

### The Scenario

Your project uses `pyproject.toml` for Python packaging. You want to keep all configuration in one file instead of having separate `pytest.ini`.

Pytest supports three configuration formats:

1. **pytest.ini** - Dedicated pytest configuration
2. **setup.cfg** - Legacy Python packaging format
3. **pyproject.toml** - Modern Python packaging format (PEP 518)

### Option 1: pytest.ini (Already Covered)

```ini
# pytest.ini
[pytest]
markers =
    unit: Fast unit tests with no external dependencies
    integration: Integration tests that use test database
```

**Pros**:

- Dedicated pytest configuration
- Clear separation of concerns
- Easy to find

**Cons**:

- Another file to maintain
- Not used if project has `pyproject.toml`

### Option 2: setup.cfg

```ini
# setup.cfg
[tool:pytest]
markers =
    unit: Fast unit tests with no external dependencies
    integration: Integration tests that use test database
```

**Pros**:

- Combines with other Python tool configuration
- Legacy projects already have this file

**Cons**:

- `setup.cfg` is being phased out in favor of `pyproject.toml`
- Less clear than dedicated `pytest.ini`

### Option 3: pyproject.toml (Recommended for New Projects)

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = [
    "unit: Fast unit tests with no external dependencies",
    "integration: Integration tests that use test database",
    "security: Security-related tests",
    "slow: Tests that take more than 1 second",
]
```

**Pros**:

- Modern Python standard (PEP 518)
- Single file for all tool configuration
- Better for projects using Poetry, Hatch, or PDM

**Cons**:

- TOML syntax slightly different from INI
- Requires pytest 6.0+

### Which Format to Choose?

**Decision tree**:

1. **Does your project have `pyproject.toml`?**
   - Yes → Use `pyproject.toml`
   - No → Continue to step 2

2. **Are you starting a new project?**
   - Yes → Use `pyproject.toml` (future-proof)
   - No → Continue to step 3

3. **Does your project have `setup.cfg`?**
   - Yes → Use `setup.cfg` (consistency)
   - No → Use `pytest.ini` (clarity)

**For our payment processor example**, we'll use `pyproject.toml` since it's a modern project:

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "payment-processor"
version = "1.0.0"
dependencies = [
    "pytest>=7.0",
]

[tool.pytest.ini_options]
markers = [
    "unit: Fast unit tests with no external dependencies",
    "integration: Integration tests that use test database",
    "e2e: End-to-end tests that test complete workflows",
    "security: Security-related tests (fraud detection, PCI compliance)",
    "performance: Performance and load testing",
    "smoke: Critical path tests that should always pass",
    "slow: Tests that take more than 1 second",
    "fast: Tests that complete in under 100ms",
    "requires_database: Test requires database connection",
    "requires_network: Test requires network access",
    "requires_auth: Test requires authentication setup",
]
```

## Iteration 3: Marker Descriptions and Documentation

### The Scenario

A new developer joins your team. They see `@pytest.mark.smoke` in the code and wonder:

- What does "smoke" mean?
- When should they use it?
- What's the difference between `smoke` and `fast`?

**The marker description should answer these questions.**

### Writing Good Marker Descriptions

**Bad descriptions** (too vague):

```toml
markers = [
    "unit: Unit tests",
    "slow: Slow tests",
    "security: Security tests",
]
```

**Good descriptions** (clear and actionable):

```toml
markers = [
    "unit: Fast unit tests with no external dependencies (< 100ms, no database/network)",
    "slow: Tests that take more than 1 second (typically integration or e2e tests)",
    "security: Security-related tests including fraud detection, PCI compliance, and vulnerability scanning",
]
```

### Comprehensive Marker Documentation

Here's a complete, well-documented marker configuration:

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = [
    # Test Types (mutually exclusive - each test should have exactly one)
    "unit: Fast unit tests with no external dependencies (< 100ms, no database/network)",
    "integration: Integration tests that use test database or external services (1-5 seconds)",
    "e2e: End-to-end tests that test complete user workflows (5+ seconds)",
    
    # Speed Categories (optional, for additional filtering)
    "fast: Tests that complete in under 100ms (useful for TDD workflow)",
    "slow: Tests that take more than 1 second (run less frequently)",
    
    # Functional Categories (optional, can have multiple per test)
    "security: Security-related tests (fraud detection, PCI compliance, vulnerability scanning)",
    "performance: Performance and load testing (benchmarks, stress tests)",
    "smoke: Critical path tests that should always pass (run first in CI)",
    
    # Requirements (optional, documents what test needs)
    "requires_database: Test requires database connection (uses test_database fixture)",
    "requires_network: Test requires network access (may fail if offline)",
    "requires_auth: Test requires authentication setup (uses auth fixtures)",
    
    # Platform-Specific (optional, combined with skipif)
    "linux_only: Test only runs on Linux (uses Linux-specific features)",
    "windows_only: Test only runs on Windows (uses Windows-specific features)",
]
```

**What makes this good**:

1. **Grouped by category**: Test types, speed, functional, requirements, platform
2. **Clear expectations**: Timing, dependencies, and usage documented
3. **Mutually exclusive noted**: "each test should have exactly one" for test types
4. **Examples provided**: "(uses test_database fixture)"
5. **Comments explain structure**: Helps maintainers understand organization

## Iteration 4: Enforcing Marker Registration

### The Scenario

A developer adds a new marker `@pytest.mark.wip` (work in progress) but forgets to register it. Tests run fine locally, but CI fails with warnings.

**You want to**: Catch unregistered markers immediately, not in CI.

### The Solution: Strict Marker Checking

Add this to your pytest configuration:

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = [
    "unit: Fast unit tests with no external dependencies",
    "integration: Integration tests that use test database",
    # ... other markers ...
]

# Treat unknown markers as errors
addopts = "--strict-markers"
```

**Now try using an unregistered marker**:

```python
# test_payment_processor.py
import pytest

@pytest.mark.wip  # Not registered!
def test_new_feature():
    pass
```

```bash
$ pytest test_payment_processor.py::test_new_feature
======================== test session starts ========================
collected 0 items / 1 error

================================ ERRORS ================================
_____________ ERROR collecting test_payment_processor.py ______________
'wip' not found in `markers` configuration option
```

**What happened**:

1. ❌ Test collection failed (didn't even run tests)
2. ❌ Clear error message: "'wip' not found in `markers` configuration option"
3. ✅ Caught the mistake immediately

**To fix**: Register the marker:

```toml
[tool.pytest.ini_options]
markers = [
    "unit: Fast unit tests with no external dependencies",
    "integration: Integration tests that use test database",
    "wip: Work in progress - test under development",
]
addopts = "--strict-markers"
```

**When to use strict markers**:

- ✅ Team projects (prevent typos)
- ✅ CI/CD pipelines (catch mistakes early)
- ✅ Projects with many markers (enforce discipline)
- ❌ Solo projects (may be too strict)
- ❌ Exploratory testing (slows down experimentation)

## Iteration 5: Configuration Inheritance and Overrides

### The Scenario

Your payment processor is part of a larger monorepo with multiple Python projects. Each project has its own markers, but some markers are shared across all projects.

**Project structure**:

```bash
monorepo/
├── pyproject.toml          # Root configuration (shared markers)
├── payment-processor/
│   ├── pyproject.toml      # Payment-specific markers
│   └── tests/
├── fraud-detection/
│   ├── pyproject.toml      # Fraud-specific markers
│   └── tests/
└── reporting/
    ├── pyproject.toml      # Reporting-specific markers
    └── tests/
```

### Root Configuration (Shared Markers)

```toml
# monorepo/pyproject.toml
[tool.pytest.ini_options]
markers = [
    # Shared markers used by all projects
    "unit: Fast unit tests with no external dependencies",
    "integration: Integration tests that use test database",
    "e2e: End-to-end tests",
    "slow: Tests that take more than 1 second",
    "fast: Tests that complete in under 100ms",
]
```

### Project-Specific Configuration

```toml
# monorepo/payment-processor/pyproject.toml
[tool.pytest.ini_options]
markers = [
    # Payment-specific markers (in addition to shared markers)
    "pci_compliance: PCI DSS compliance tests",
    "payment_gateway: Tests that interact with payment gateway",
    "refund: Refund processing tests",
]
```

```toml
# monorepo/fraud-detection/pyproject.toml
[tool.pytest.ini_options]
markers = [
    # Fraud-specific markers
    "ml_model: Tests for machine learning models",
    "rule_engine: Tests for fraud detection rules",
    "false_positive: Tests for false positive scenarios",
]
```

**How pytest resolves configuration**:

1. Looks for configuration in current directory
2. Walks up directory tree until it finds a configuration file
3. Uses the **first** configuration file found (no merging)

**Problem**: Project-specific markers override shared markers (no inheritance).

### Solution: Document Shared Markers in Each Project

Since pytest doesn't merge configurations, you need to duplicate shared markers:

```toml
# monorepo/payment-processor/pyproject.toml
[tool.pytest.ini_options]
markers = [
    # Shared markers (duplicated from root)
    "unit: Fast unit tests with no external dependencies",
    "integration: Integration tests that use test database",
    "e2e: End-to-end tests",
    "slow: Tests that take more than 1 second",
    "fast: Tests that complete in under 100ms",
    
    # Payment-specific markers
    "pci_compliance: PCI DSS compliance tests",
    "payment_gateway: Tests that interact with payment gateway",
    "refund: Refund processing tests",
]
```

**Alternative**: Use a shared configuration file:

```python
# monorepo/shared_pytest_config.py
SHARED_MARKERS = [
    "unit: Fast unit tests with no external dependencies",
    "integration: Integration tests that use test database",
    "e2e: End-to-end tests",
    "slow: Tests that take more than 1 second",
    "fast: Tests that complete in under 100ms",
]
```

```python
# monorepo/payment-processor/conftest.py
import sys
sys.path.insert(0, "..")
from shared_pytest_config import SHARED_MARKERS

def pytest_configure(config):
    """Register shared markers programmatically."""
    for marker in SHARED_MARKERS:
        config.addinivalue_line("markers", marker)
    
    # Add project-specific markers
    config.addinivalue_line("markers", "pci_compliance: PCI DSS compliance tests")
    config.addinivalue_line("markers", "payment_gateway: Tests that interact with payment gateway")
```

**This approach**:

- ✅ Single source of truth for shared markers
- ✅ Each project can add its own markers
- ✅ No duplication
- ❌ More complex (requires conftest.py)

## Diagnostic Analysis: Configuration Issues

### Issue 1: Markers Not Recognized Despite Registration

**Symptom**: Markers registered in `pyproject.toml` but pytest still warns about unknown markers.

```bash
$ pytest
PytestUnknownMarkWarning: Unknown pytest.mark.unit
```

**Diagnostic steps**:

1. **Check pytest version**: `pytest --version`
   - `pyproject.toml` support requires pytest 6.0+
   - If using older pytest, use `pytest.ini` instead

2. **Verify configuration location**: `pytest --version -v`
   - Pytest shows which configuration file it's using
   - Make sure it's finding your `pyproject.toml`

3. **Check TOML syntax**:

```toml
# Wrong (missing quotes)
markers = [
    unit: Fast unit tests
]

# Correct
markers = [
    "unit: Fast unit tests",
]
```

4. **Verify section name**:

```toml
# Wrong
[pytest]
markers = [...]

# Correct
[tool.pytest.ini_options]
markers = [...]
```

### Issue 2: Configuration Not Found

**Symptom**: Pytest doesn't find your configuration file.

```bash
$ pytest --version -v
pytest 7.4.0
rootdir: /home/user/project
configfile: None  # <-- No config file found!
```

**Root cause**: Configuration file not in pytest's search path.

**Pytest searches for configuration in this order**:

1. `pytest.ini` in current directory
2. `pyproject.toml` in current directory
3. `setup.cfg` in current directory
4. Walk up directory tree repeating steps 1-3

**Solution**: Place configuration file in project root or run pytest from correct directory.

### Issue 3: Strict Markers Too Strict

**Symptom**: Can't use built-in markers with `--strict-markers`.

```bash
$ pytest --strict-markers
ERROR: 'parametrize' not found in `markers` configuration option
```

**Root cause**: `--strict-markers` requires ALL markers to be registered, including built-in ones.

**Solution**: Don't register built-in markers—they're automatically available. The error suggests you're using a marker that looks built-in but isn't, or there's a typo.

**Check**: `pytest --markers` to see all available markers.

## Configuration Best Practices

### 1. Choose One Configuration Format

Don't mix formats. If you have `pyproject.toml`, don't also create `pytest.ini`.

**Pytest's precedence** (first found wins):

1. `pytest.ini`
2. `pyproject.toml`
3. `tox.ini`
4. `setup.cfg`

### 2. Document Marker Usage

Include usage guidelines in your marker descriptions:

```toml
markers = [
    "unit: Fast unit tests (< 100ms). Use for: pure functions, business logic, no I/O",
    "integration: Integration tests (1-5s). Use for: database operations, API calls",
]
```

### 3. Use Strict Markers in CI

Enable strict markers in CI but not locally:

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = [...]

# Don't add --strict-markers here
# Instead, add it in CI configuration
```

```yaml
# .github/workflows/test.yml
- name: Run tests
  run: pytest --strict-markers
```

**Why**: Developers can experiment locally, but CI enforces discipline.

### 4. Keep Marker List Organized

Group markers by category with comments:

```toml
markers = [
    # === Test Types ===
    "unit: ...",
    "integration: ...",
    
    # === Speed ===
    "fast: ...",
    "slow: ...",
    
    # === Requirements ===
    "requires_database: ...",
]
```

### 5. Review Markers Regularly

Markers accumulate over time. Periodically review and remove unused markers:

```bash
# Find all markers used in code
$ grep -r "@pytest.mark." tests/ | sed 's/.*@pytest.mark.\([a-z_]*\).*/\1/' | sort -u

# Compare with registered markers
$ pytest --markers | grep "^@pytest.mark" | grep -v "skip\|xfail\|parametrize"
```

## What You've Learned

Marker registration provides a central source of truth for test categorization:

1. **Three configuration formats**: `pytest.ini`, `setup.cfg`, `pyproject.toml`
2. **Choose `pyproject.toml`** for new projects (modern standard)
3. **Write clear descriptions** that explain when to use each marker
4. **Use `--strict-markers`** to catch typos and enforce registration
5. **Configuration doesn't inherit** in monorepos (must duplicate or use conftest.py)

**Key insights**:

- Registration prevents typos and documents intent
- Good descriptions answer "when should I use this marker?"
- Strict markers enforce discipline in team projects
- Configuration format choice depends on project structure

**What's next**: We've registered markers and applied them to tests. Now we need to use them effectively. The next section covers filtering tests by markers and building efficient test execution strategies.

## Filtering Tests by Markers

## The Problem: Running the Right Tests at the Right Time

Your payment processor test suite has 150 tests:

- 80 fast unit tests (< 100ms each)
- 50 integration tests (1-3 seconds each)
- 15 end-to-end tests (5-10 seconds each)
- 5 performance tests (30+ seconds each)

**Total runtime**: ~8 minutes for the full suite.

**The challenge**: Different contexts need different test subsets:

- **During development**: Run only fast unit tests (instant feedback)
- **Before committing**: Run unit + integration tests (confidence)
- **In CI pull requests**: Run everything except performance tests (thorough but fast)
- **Nightly builds**: Run everything including performance tests (comprehensive)

**What you need**: Precise control over which tests run in each context.

## Iteration 1: Basic Marker Filtering

### The Scenario

You're developing a new feature in the payment validation module. You want to run only the unit tests for this module to get instant feedback.

**First, let's see all tests**:

```bash
$ pytest --collect-only -q
test_payment_processor.py::test_validate_credit_card_accepts_valid
test_payment_processor.py::test_validate_credit_card_rejects_invalid
test_payment_processor.py::test_calculate_transaction_fee
test_payment_processor.py::test_process_payment_records_transaction
test_payment_processor.py::test_refund_updates_transaction_status
test_payment_processor.py::test_complete_payment_workflow
test_fraud_detection.py::test_fraud_score_calculation
test_fraud_detection.py::test_fraud_detection_blocks_suspicious
test_reporting.py::test_generate_transaction_report
test_reporting.py::test_generate_annual_report

10 tests collected
```

### Filter by Single Marker

Run only unit tests:

```bash
$ pytest -m unit -v
======================== test session starts ========================
collected 10 items / 6 deselected / 4 selected

test_payment_processor.py::test_validate_credit_card_accepts_valid PASSED
test_payment_processor.py::test_validate_credit_card_rejects_invalid PASSED
test_payment_processor.py::test_calculate_transaction_fee PASSED
test_fraud_detection.py::test_fraud_score_calculation PASSED

=================== 4 passed, 6 deselected in 0.08s ===================
```

**What happened**:

1. **"10 items / 6 deselected / 4 selected"**: Pytest found 10 tests, filtered out 6, ran 4
2. **Execution time**: 0.08 seconds (vs. 8 minutes for full suite)
3. **Only unit tests ran**: Integration and e2e tests were skipped

### Understanding the -m Flag

The `-m` flag accepts a **marker expression**:

```bash
# Basic syntax
pytest -m MARKER_NAME

# Examples
pytest -m unit           # Run tests marked with @pytest.mark.unit
pytest -m integration    # Run tests marked with @pytest.mark.integration
pytest -m slow           # Run tests marked with @pytest.mark.slow
```

**Key insight**: The `-m` flag doesn't run tests in a specific file or directory—it filters the entire test collection based on markers.

## Iteration 2: Boolean Logic in Marker Expressions

### The Scenario

You want to run unit tests AND integration tests, but not e2e tests (for pre-commit hook).

### Using OR Logic

Run tests that are EITHER unit OR integration:

```bash
$ pytest -m "unit or integration" -v
======================== test session starts ========================
collected 10 items / 2 deselected / 8 selected

test_payment_processor.py::test_validate_credit_card_accepts_valid PASSED
test_payment_processor.py::test_validate_credit_card_rejects_invalid PASSED
test_payment_processor.py::test_calculate_transaction_fee PASSED
test_payment_processor.py::test_process_payment_records_transaction PASSED
test_payment_processor.py::test_refund_updates_transaction_status PASSED
test_fraud_detection.py::test_fraud_score_calculation PASSED
test_fraud_detection.py::test_fraud_detection_blocks_suspicious PASSED
test_reporting.py::test_generate_transaction_report PASSED

=================== 8 passed, 2 deselected in 2.45s ===================
```

**Result**: 8 tests ran (4 unit + 4 integration), 2 deselected (e2e tests).

### Using AND Logic

Run tests that are BOTH integration AND security:

```bash
$ pytest -m "integration and security" -v
======================== test session starts ========================
collected 10 items / 9 deselected / 1 selected

test_fraud_detection.py::test_fraud_detection_blocks_suspicious PASSED

=================== 1 passed, 9 deselected in 1.23s ===================
```

**Result**: Only 1 test has both markers.

### Using NOT Logic

Run all tests EXCEPT slow ones:

```bash
$ pytest -m "not slow" -v
======================== test session starts ========================
collected 10 items / 3 deselected / 7 selected

test_payment_processor.py::test_validate_credit_card_accepts_valid PASSED
test_payment_processor.py::test_validate_credit_card_rejects_invalid PASSED
test_payment_processor.py::test_calculate_transaction_fee PASSED
test_payment_processor.py::test_process_payment_records_transaction PASSED
test_fraud_detection.py::test_fraud_score_calculation PASSED
test_fraud_detection.py::test_fraud_detection_blocks_suspicious PASSED
test_reporting.py::test_generate_transaction_report PASSED

=================== 7 passed, 3 deselected in 2.15s ===================
```

**Result**: 7 tests ran (all except the 3 marked as slow).

## Iteration 3: Complex Marker Expressions

### The Scenario

Your CI pipeline has different stages:

1. **Fast feedback**: Unit tests only
2. **Integration check**: Integration tests that don't require network
3. **Security scan**: All security tests
4. **Full suite**: Everything except performance tests

Let's build marker expressions for each stage.

### Stage 1: Fast Feedback (Unit Tests Only)

```bash
# Simple: just unit tests
$ pytest -m unit

# More precise: unit tests that are also fast
$ pytest -m "unit and fast"
```

### Stage 2: Integration Tests Without Network

```bash
# Integration tests that don't require network
$ pytest -m "integration and not requires_network"
```

**Example test that matches**:

```python
@pytest.mark.integration
@pytest.mark.requires_database
def test_process_payment_records_transaction(test_database):
    """This test runs (has integration, no requires_network)."""
    pass
```

**Example test that doesn't match**:

```python
@pytest.mark.integration
@pytest.mark.requires_network
def test_payment_gateway_integration():
    """This test is skipped (has requires_network)."""
    pass
```

### Stage 3: All Security Tests

```bash
# All security tests regardless of type
$ pytest -m security
```

### Stage 4: Full Suite Except Performance

```bash
# Everything except performance tests
$ pytest -m "not performance"

# More explicit: unit, integration, or e2e, but not performance
$ pytest -m "(unit or integration or e2e) and not performance"
```

### Operator Precedence

Marker expressions follow boolean logic precedence:

1. **`not`** (highest precedence)
2. **`and`**
3. **`or`** (lowest precedence)

**Example without parentheses**:

```bash
# This expression:
pytest -m "unit or integration and not slow"

# Is evaluated as:
pytest -m "unit or (integration and (not slow))"

# Meaning: Run tests that are EITHER:
# - unit tests (any speed), OR
# - integration tests that are not slow
```

**Example with parentheses for clarity**:

```bash
# What you probably meant:
pytest -m "(unit or integration) and not slow"

# Meaning: Run tests that are:
# - (unit OR integration) AND not slow
```

**Best practice**: Use parentheses for complex expressions to make intent clear.

## Iteration 4: Combining Markers with Other Filters

### The Scenario

You want to run unit tests, but only for the payment processor module (not fraud detection or reporting).

### Combining -m with -k (Keyword Filter)

```bash
# Run unit tests with "payment" in the test name
$ pytest -m unit -k payment -v
======================== test session starts ========================
collected 10 items / 8 deselected / 2 selected

test_payment_processor.py::test_validate_credit_card_accepts_valid PASSED
test_payment_processor.py::test_calculate_transaction_fee PASSED

=================== 2 passed, 8 deselected in 0.05s ===================
```

**What happened**:

1. First filter: `-m unit` selected 4 tests (all unit tests)
2. Second filter: `-k payment` selected tests with "payment" in name
3. Result: 2 tests matched both filters

### Combining -m with File/Directory Selection

```bash
# Run unit tests only in test_payment_processor.py
$ pytest -m unit test_payment_processor.py -v
======================== test session starts ========================
collected 6 items / 3 deselected / 3 selected

test_payment_processor.py::test_validate_credit_card_accepts_valid PASSED
test_payment_processor.py::test_validate_credit_card_rejects_invalid PASSED
test_payment_processor.py::test_calculate_transaction_fee PASSED

=================== 3 passed, 3 deselected in 0.05s ===================
```

**What happened**:

1. First filter: `test_payment_processor.py` selected 6 tests from that file
2. Second filter: `-m unit` selected only unit tests
3. Result: 3 tests matched both filters

### Combining Multiple Filters

You can combine all filter types:

```bash
# Run fast unit tests with "validate" in name from payment processor file
$ pytest -m "unit and fast" -k validate test_payment_processor.py -v
======================== test session starts ========================
collected 6 items / 4 deselected / 2 selected

test_payment_processor.py::test_validate_credit_card_accepts_valid PASSED
test_payment_processor.py::test_validate_credit_card_rejects_invalid PASSED

=================== 2 passed, 4 deselected in 0.03s ===================
```

**Filter order doesn't matter**: Pytest applies all filters to the collected tests.

## Iteration 5: Practical Filtering Strategies

### Development Workflow

Create shell aliases or scripts for common filter combinations:

```bash
# ~/.bashrc or ~/.zshrc

# Fast feedback during development
alias pytest-fast="pytest -m 'unit and fast'"

# Pre-commit checks
alias pytest-commit="pytest -m 'unit or integration'"

# Security audit
alias pytest-security="pytest -m security"

# Full suite except slow tests
alias pytest-quick="pytest -m 'not slow'"
```

**Usage**:

```bash
$ pytest-fast
# Runs only fast unit tests

$ pytest-commit
# Runs unit and integration tests
```

### CI/CD Pipeline Configuration

Define marker expressions in your CI configuration:

```yaml
# .github/workflows/test.yml
name: Test Suite

on: [push, pull_request]

jobs:
  fast-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run fast tests
        run: pytest -m "unit and fast" --tb=short
  
  integration-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run integration tests
        run: pytest -m "integration and not requires_network"
  
  security-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run security tests
        run: pytest -m security
  
  full-suite:
    runs-on: ubuntu-latest
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v3
      - name: Run full test suite
        run: pytest -m "not performance"
```

**This configuration**:

1. **Fast tests**: Run on every push (instant feedback)
2. **Integration tests**: Run in parallel (faster CI)
3. **Security tests**: Run separately (clear reporting)
4. **Full suite**: Run only on main branch (comprehensive but slow)

### Make Configuration

Create a Makefile for common test commands:

```makefile
# Makefile

.PHONY: test test-fast test-unit test-integration test-security test-all

# Fast feedback during development
test-fast:
	pytest -m "unit and fast" -v

# Unit tests only
test-unit:
	pytest -m unit -v

# Integration tests only
test-integration:
	pytest -m integration -v

# Security tests
test-security:
	pytest -m security -v

# Pre-commit checks
test-commit:
	pytest -m "unit or integration" -v

# Full suite
test-all:
	pytest -v

# Default target
test: test-fast
```

**Usage**:

```bash
$ make test          # Runs fast tests (default)
$ make test-unit     # Runs unit tests
$ make test-commit   # Runs pre-commit checks
$ make test-all      # Runs everything
```

## Diagnostic Analysis: Filtering Issues

### Issue 1: No Tests Selected

**Symptom**: Marker filter selects zero tests.

```bash
$ pytest -m "unit and security"
======================== test session starts ========================
collected 10 items / 10 deselected / 0 selected

=================== 10 deselected in 0.02s ===================
```

**Diagnostic steps**:

1. **Check if any tests have both markers**:

```bash
# List all tests with their markers
$ pytest --collect-only -m unit
$ pytest --collect-only -m security
```

2. **Verify marker names**:

```bash
# Check registered markers
$ pytest --markers | grep -E "unit|security"
```

3. **Check for typos**:

```bash
# This won't match anything (typo: "secuirty")
$ pytest -m "unit and secuirty"
```

**Root cause**: Either:

- No tests have both markers (use `or` instead of `and`)
- Typo in marker name
- Markers not applied to tests

### Issue 2: Unexpected Tests Selected

**Symptom**: Marker filter selects more tests than expected.

```bash
$ pytest -m "unit or integration and not slow"
# Selects more tests than you expected
```

**Root cause**: Operator precedence.

**What you wrote**:

```bash
pytest -m "unit or integration and not slow"
```

**How pytest interprets it**:

```bash
pytest -m "unit or (integration and (not slow))"
```

**What you probably meant**:

```bash
pytest -m "(unit or integration) and not slow"
```

**Solution**: Use parentheses to make precedence explicit.

### Issue 3: Marker Expression Syntax Error

**Symptom**: Pytest reports syntax error in marker expression.

```bash
$ pytest -m "unit && integration"
ERROR: Wrong expression passed to '-m': unit && integration
```

**Root cause**: Using wrong boolean operators.

**Wrong operators** (from other languages):

- `&&` (C/Java/JavaScript)
- `||` (C/Java/JavaScript)
- `!` (C/Java/JavaScript)

**Correct operators** (Python-style):

- `and`
- `or`
- `not`

**Solution**: Use Python boolean operators:

```bash
# Correct
$ pytest -m "unit and integration"
$ pytest -m "unit or integration"
$ pytest -m "not slow"
```

## Advanced Filtering Techniques

### Technique 1: Marker Expressions in pytest.ini

Define default marker expressions in configuration:

```toml
# pyproject.toml
[tool.pytest.ini_options]
markers = [
    "unit: Fast unit tests",
    "integration: Integration tests",
    "slow: Slow tests",
]

# Default: run everything except slow tests
addopts = "-m 'not slow'"
```

**Now**:

```bash
# This runs "not slow" by default
$ pytest

# Override default to run everything
$ pytest -m ""

# Override default to run only slow tests
$ pytest -m slow
```

### Technique 2: Custom Marker Expressions via Environment Variables

```bash
# Set marker expression via environment variable
export PYTEST_MARKERS="unit and fast"

# Use in pytest command
pytest -m "$PYTEST_MARKERS"
```

**Use case**: Different CI environments can set different marker expressions.

### Technique 3: Programmatic Marker Filtering

Use pytest hooks to filter tests programmatically:

```python
# conftest.py
import os
import pytest

def pytest_collection_modifyitems(config, items):
    """Filter tests based on environment variable."""
    marker_expr = os.environ.get("TEST_MARKERS")
    
    if marker_expr:
        # Parse marker expression
        selected = []
        deselected = []
        
        for item in items:
            # Evaluate marker expression against test's markers
            if should_run_test(item, marker_expr):
                selected.append(item)
            else:
                deselected.append(item)
        
        # Modify items list
        items[:] = selected
        
        # Report deselection
        config.hook.pytest_deselected(items=deselected)

def should_run_test(item, marker_expr):
    """Evaluate if test should run based on marker expression."""
    # This is simplified - real implementation would parse the expression
    markers = {mark.name for mark in item.iter_markers()}
    
    # Example: "unit and fast"
    if marker_expr == "unit and fast":
        return "unit" in markers and "fast" in markers
    
    # Add more expression parsing as needed
    return True
```

**Usage**:

```bash
# Set marker expression via environment
export TEST_MARKERS="unit and fast"

# Run tests (hook filters automatically)
pytest
```

**Note**: This is advanced usage. For most cases, use `-m` flag directly.

## Filtering Best Practices

### 1. Start Broad, Then Narrow

When debugging, start with broad filters and progressively narrow:

```bash
# Step 1: Run all tests to see failures
$ pytest

# Step 2: Run only failing test type
$ pytest -m integration

# Step 3: Run only failing test file
$ pytest -m integration test_payment_processor.py

# Step 4: Run only failing test
$ pytest test_payment_processor.py::test_process_payment
```

### 2. Use Descriptive Marker Names

**Bad** (ambiguous):

```python
@pytest.mark.fast
@pytest.mark.db
def test_something():
    pass
```

**Good** (clear):

```python
@pytest.mark.unit
@pytest.mark.requires_database
def test_something():
    pass
```

### 3. Document Common Filter Expressions

Create a README or Makefile documenting common filters:

```markdown
# Testing Guide

## Common Test Commands

### Development
```bash
# Fast feedback (< 1 second)
pytest -m "unit and fast"

# Pre-commit checks (< 5 seconds)
pytest -m "unit or integration"
```

### CI/CD
```bash
# Pull request checks
pytest -m "not performance"

# Nightly builds
pytest  # Run everything
```

### Debugging
```bash
# Run only failing test type
pytest -m integration --lf  # --lf = last failed

# Run tests for specific module
pytest -m unit -k payment
```
```

### 4. Avoid Over-Filtering

**Anti-pattern**: Too many marker combinations.

```bash
# This is too specific
pytest -m "unit and fast and not requires_database and not requires_network and security"
```

**Better**: Simplify marker taxonomy.

```bash
# Simpler and clearer
pytest -m "unit and security"
```

**Guideline**: If you need more than 3 boolean operators, your marker taxonomy is too complex.

## What You've Learned

Marker filtering provides precise control over test execution:

1. **Basic filtering**: `-m marker_name` runs tests with that marker
2. **Boolean logic**: Use `and`, `or`, `not` to combine markers
3. **Operator precedence**: `not` > `and` > `or` (use parentheses for clarity)
4. **Combine filters**: Mix `-m` with `-k` and file selection
5. **Practical strategies**: Shell aliases, CI configuration, Makefiles

**Key insights**:

- Marker expressions are evaluated against each test's markers
- Complex expressions need parentheses for clarity
- Filtering is composable (multiple filters combine)
- Good marker taxonomy makes filtering intuitive

**What's next**: We've learned to filter tests by markers. The final section covers organizing entire test suites using markers—building a comprehensive testing strategy that scales from solo development to large teams.

## Organizing Tests by Category

## The Problem: Test Suite Chaos at Scale

Your payment processor has grown to 500+ tests across 50 files. The test suite has become difficult to navigate:

- New developers don't know where to add tests
- Related tests are scattered across multiple files
- No clear testing strategy or standards
- CI runs take 30+ minutes
- Flaky tests hide real failures

**What you need**: A systematic approach to organizing tests that:

1. Makes test structure intuitive
2. Enables efficient test execution
3. Scales with team size
4. Maintains test quality

This is what marker-based test organization solves.

## Iteration 1: Establishing a Test Organization Strategy

### The Scenario

You're restructuring your test suite. You need to decide:

- How to categorize tests
- How to structure test files
- How to use markers effectively
- How to document the organization

### Step 1: Define Your Test Taxonomy

Start by identifying the dimensions that matter for your project:

```markdown
# Test Organization Strategy

## Primary Dimensions

### 1. Test Type (Mutually Exclusive)
- **unit**: Isolated, fast, no external dependencies
- **integration**: Tests component interactions (database, APIs)
- **e2e**: Full user workflows, slowest

### 2. Test Speed (Optional)
- **fast**: < 100ms (for TDD workflow)
- **slow**: > 1 second (run less frequently)

### 3. Functional Category (Multiple Allowed)
- **security**: Fraud detection, PCI compliance, vulnerability testing
- **performance**: Benchmarks, load tests, stress tests
- **smoke**: Critical path tests that must always pass

### 4. Requirements (Multiple Allowed)
- **requires_database**: Needs database connection
- **requires_network**: Needs network access
- **requires_auth**: Needs authentication setup

## Marker Application Rules

1. Every test MUST have exactly one test type marker (unit/integration/e2e)
2. Tests SHOULD have a speed marker if they're notably fast or slow
3. Tests MAY have functional category markers
4. Tests SHOULD have requirement markers to document dependencies
```

### Step 2: Structure Test Files by Feature

Organize test files to mirror your application structure:

```bash
payment-processor/
├── src/
│   └── payment_processor/
│       ├── __init__.py
│       ├── validation.py      # Credit card validation
│       ├── processing.py      # Payment processing
│       ├── fraud.py           # Fraud detection
│       └── reporting.py       # Transaction reporting
│
└── tests/
    ├── conftest.py            # Shared fixtures
    ├── unit/                  # Unit tests (fast, isolated)
    │   ├── test_validation.py
    │   ├── test_processing.py
    │   ├── test_fraud.py
    │   └── test_reporting.py
    │
    ├── integration/           # Integration tests (database, APIs)
    │   ├── test_payment_flow.py
    │   ├── test_fraud_detection.py
    │   └── test_reporting_queries.py
    │
    ├── e2e/                   # End-to-end tests (full workflows)
    │   ├── test_checkout_flow.py
    │   └── test_refund_flow.py
    │
    └── performance/           # Performance tests (benchmarks)
        └── test_payment_benchmarks.py
```

**Benefits of this structure**:

1. **Clear separation**: Test type is obvious from directory
2. **Easy navigation**: Find tests by feature or type
3. **Parallel execution**: Can run directories in parallel
4. **Selective execution**: Can run entire directories

### Step 3: Apply Markers Consistently

Create a template for each test type:

```python
# tests/unit/test_validation.py
"""
Unit tests for credit card validation.

All tests in this file should be:
- Marked as @pytest.mark.unit
- Marked as @pytest.mark.fast (they should be < 100ms)
- Have no external dependencies
"""
import pytest
from payment_processor.validation import validate_credit_card, validate_cvv

@pytest.mark.unit
@pytest.mark.fast
def test_validate_credit_card_accepts_valid_visa():
    """Validate Visa card using Luhn algorithm."""
    valid_visa = "4532015112830366"
    assert validate_credit_card(valid_visa) is True

@pytest.mark.unit
@pytest.mark.fast
def test_validate_credit_card_rejects_invalid_checksum():
    """Reject card with invalid Luhn checksum."""
    invalid_card = "4532015112830367"  # Last digit wrong
    assert validate_credit_card(invalid_card) is False

@pytest.mark.unit
@pytest.mark.fast
def test_validate_cvv_accepts_three_digits():
    """Accept valid 3-digit CVV."""
    assert validate_cvv("123") is True

@pytest.mark.unit
@pytest.mark.fast
def test_validate_cvv_rejects_non_numeric():
    """Reject CVV with non-numeric characters."""
    assert validate_cvv("12A") is False
```

```python
# tests/integration/test_payment_flow.py
"""
Integration tests for payment processing flow.

All tests in this file should be:
- Marked as @pytest.mark.integration
- Marked as @pytest.mark.slow (they use database)
- Marked as @pytest.mark.requires_database
- Use test_database fixture
"""
import pytest
from payment_processor.processing import process_payment, refund_payment

@pytest.mark.integration
@pytest.mark.slow
@pytest.mark.requires_database
def test_process_payment_records_transaction(test_database):
    """Process payment and verify database record created."""
    result = process_payment(
        amount=100.00,
        card_number="4532015112830366",
        database=test_database
    )
    
    assert result["status"] == "success"
    
    # Verify transaction recorded
    cursor = test_database.execute(
        "SELECT * FROM transactions WHERE amount = ?", (100.00,)
    )
    transaction = cursor.fetchone()
    assert transaction is not None
    assert transaction["status"] == "completed"

@pytest.mark.integration
@pytest.mark.slow
@pytest.mark.requires_database
def test_refund_updates_transaction_status(test_database):
    """Process refund and verify status updated in database."""
    # First create a transaction
    payment_result = process_payment(
        amount=100.00,
        card_number="4532015112830366",
        database=test_database
    )
    transaction_id = payment_result["transaction_id"]
    
    # Then refund it
    refund_result = refund_payment(
        transaction_id=transaction_id,
        database=test_database
    )
    
    assert refund_result["status"] == "success"
    
    # Verify status updated
    cursor = test_database.execute(
        "SELECT status FROM transactions WHERE id = ?", (transaction_id,)
    )
    status = cursor.fetchone()["status"]
    assert status == "refunded"

@pytest.mark.integration
@pytest.mark.slow
@pytest.mark.requires_database
@pytest.mark.security
def test_fraud_detection_blocks_suspicious_pattern(test_database):
    """Verify fraud detection blocks suspicious transaction patterns."""
    card_number = "4532015112830366"
    
    # Attempt 10 rapid transactions (suspicious pattern)
    for i in range(10):
        result = process_payment(
            amount=1000.00,
            card_number=card_number,
            database=test_database
        )
    
    # 10th transaction should be blocked by fraud detection
    assert result["status"] == "blocked"
    assert result["reason"] == "fraud_detection"
```

```python
# tests/e2e/test_checkout_flow.py
"""
End-to-end tests for complete checkout flow.

All tests in this file should be:
- Marked as @pytest.mark.e2e
- Marked as @pytest.mark.slow (full workflow)
- Marked with all required dependencies
- Test complete user workflows
"""
import pytest
from payment_processor import create_cart, checkout, confirm_order

@pytest.mark.e2e
@pytest.mark.slow
@pytest.mark.requires_database
@pytest.mark.requires_network
@pytest.mark.smoke
def test_complete_checkout_flow(test_database, mock_payment_gateway):
    """Test complete checkout flow from cart to confirmation."""
    # 1. Create shopping cart
    cart = create_cart(items=[
        {"product_id": 1, "quantity": 2, "price": 29.99},
        {"product_id": 2, "quantity": 1, "price": 49.99}
    ])
    assert cart.total == 109.97
    
    # 2. Process checkout
    checkout_result = checkout(
        cart=cart,
        card_number="4532015112830366",
        cvv="123",
        database=test_database
    )
    assert checkout_result["status"] == "success"
    
    # 3. Verify payment gateway called
    assert mock_payment_gateway.was_called_with(amount=109.97)
    
    # 4. Confirm order
    order = confirm_order(
        checkout_id=checkout_result["checkout_id"],
        database=test_database
    )
    assert order.status == "confirmed"
    assert order.total == 109.97
    
    # 5. Verify database state
    cursor = test_database.execute(
        "SELECT * FROM orders WHERE id = ?", (order.id,)
    )
    db_order = cursor.fetchone()
    assert db_order["status"] == "confirmed"
```

**Key patterns**:

1. **File-level docstring**: Explains what markers should be used
2. **Consistent marker application**: All tests in file follow same pattern
3. **Clear test names**: Describe what's being tested
4. **Appropriate markers**: Match test characteristics

## Iteration 2: Enforcing Organization Standards

### The Scenario

You want to ensure all tests follow your organization standards:

- Every test has a test type marker (unit/integration/e2e)
- Tests in `tests/unit/` are marked as unit tests
- Tests in `tests/integration/` are marked as integration tests
- No test has conflicting markers (e.g., both unit and integration)

### Solution: Pytest Hook for Validation

Create a conftest.py hook to enforce standards:

```python
# tests/conftest.py
import pytest
from pathlib import Path

def pytest_collection_modifyitems(config, items):
    """
    Enforce test organization standards:
    1. Every test must have exactly one test type marker
    2. Test type marker must match directory structure
    3. Tests with requires_* markers must not be unit tests
    """
    errors = []
    
    for item in items:
        # Get test file path
        test_path = Path(item.fspath)
        
        # Get all markers on this test
        markers = {mark.name for mark in item.iter_markers()}
        
        # Check 1: Must have exactly one test type marker
        test_type_markers = markers & {"unit", "integration", "e2e"}
        
        if len(test_type_markers) == 0:
            errors.append(
                f"{item.nodeid}: Missing test type marker (unit/integration/e2e)"
            )
        elif len(test_type_markers) > 1:
            errors.append(
                f"{item.nodeid}: Multiple test type markers: {test_type_markers}"
            )
        
        # Check 2: Test type must match directory
        if "unit" in markers and "tests/unit" not in str(test_path):
            errors.append(
                f"{item.nodeid}: Unit test not in tests/unit/ directory"
            )
        
        if "integration" in markers and "tests/integration" not in str(test_path):
            errors.append(
                f"{item.nodeid}: Integration test not in tests/integration/ directory"
            )
        
        if "e2e" in markers and "tests/e2e" not in str(test_path):
            errors.append(
                f"{item.nodeid}: E2E test not in tests/e2e/ directory"
            )
        
        # Check 3: Unit tests shouldn't have external dependencies
        if "unit" in markers:
            dependency_markers = markers & {
                "requires_database", "requires_network", "requires_auth"
            }
            if dependency_markers:
                errors.append(
                    f"{item.nodeid}: Unit test has dependency markers: {dependency_markers}"
                )
    
    # Report all errors
    if errors:
        error_message = "\n".join([
            "Test organization violations found:",
            *errors,
            "",
            "See tests/README.md for organization standards."
        ])
        pytest.exit(error_message, returncode=1)
```

**Run tests with violations**:

```bash
$ pytest
======================== test session starts ========================
Test organization violations found:
tests/unit/test_validation.py::test_validate_with_database: Unit test has dependency markers: {'requires_database'}
tests/integration/test_payment.py::test_process_payment: Missing test type marker (unit/integration/e2e)
tests/unit/test_fraud.py::test_fraud_detection: Multiple test type markers: {'unit', 'integration'}

See tests/README.md for organization standards.
```

**What this achieves**:

1. ✅ Catches organization violations immediately
2. ✅ Prevents tests from being committed with wrong markers
3. ✅ Enforces consistency across the team
4. ✅ Self-documenting (error messages explain the rules)

## Iteration 3: Building Test Execution Strategies

### The Scenario

Different contexts need different test execution strategies:

1. **Local development**: Fast feedback (< 1 second)
2. **Pre-commit hook**: Confidence before committing (< 10 seconds)
3. **CI pull request**: Thorough but fast (< 5 minutes)
4. **CI main branch**: Comprehensive (< 30 minutes)
5. **Nightly build**: Everything including performance (unlimited time)

### Strategy 1: Local Development (TDD Workflow)

**Goal**: Instant feedback while coding.

**Marker expression**:

```bash
# Run only fast unit tests
pytest -m "unit and fast"
```

**Expected runtime**: < 1 second for 100+ tests

**Create a shell alias**:

```bash
# ~/.bashrc or ~/.zshrc
alias pt="pytest -m 'unit and fast' -x"  # -x stops on first failure
```

**Usage during TDD**:

```bash
# Write test
$ vim tests/unit/test_validation.py

# Run tests (instant feedback)
$ pt

# Fix code
$ vim src/payment_processor/validation.py

# Run tests again
$ pt
```

### Strategy 2: Pre-Commit Hook

**Goal**: Catch issues before committing.

**Marker expression**:

```bash
# Run unit and integration tests, skip slow e2e
pytest -m "(unit or integration) and not slow"
```

**Expected runtime**: 5-10 seconds

**Create a git pre-commit hook**:

```bash
# .git/hooks/pre-commit
#!/bin/bash

echo "Running pre-commit tests..."

# Run unit and integration tests
pytest -m "(unit or integration) and not slow" --tb=short -q

if [ $? -ne 0 ]; then
    echo "Tests failed. Commit aborted."
    echo "Run 'pytest -m \"(unit or integration) and not slow\"' to see failures."
    exit 1
fi

echo "All tests passed!"
```

**Make it executable**:

```bash
chmod +x .git/hooks/pre-commit
```

### Strategy 3: CI Pull Request

**Goal**: Thorough testing without performance tests.

**Marker expression**:

```bash
# Run everything except performance tests
pytest -m "not performance"
```

**Expected runtime**: 3-5 minutes

**GitHub Actions configuration**:

```yaml
# .github/workflows/pull-request.yml
name: Pull Request Tests

on: [pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -e .
          pip install pytest pytest-cov
      
      - name: Run tests
        run: |
          pytest -m "not performance" \
                 --cov=payment_processor \
                 --cov-report=xml \
                 --cov-report=term-missing \
                 -v
      
      - name: Upload coverage
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
```

### Strategy 4: CI Main Branch

**Goal**: Comprehensive testing including smoke tests.

**Marker expression**:

```bash
# Run everything except performance tests, but ensure smoke tests pass first
pytest -m smoke --tb=short -x  # Stop on first smoke test failure
pytest -m "not performance"     # Then run everything else
```

**Expected runtime**: 10-30 minutes

**GitHub Actions configuration**:

```yaml
# .github/workflows/main.yml
name: Main Branch Tests

on:
  push:
    branches: [main]

jobs:
  smoke-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: pip install -e .
      - name: Run smoke tests
        run: pytest -m smoke --tb=short -x
  
  full-suite:
    needs: smoke-tests  # Only run if smoke tests pass
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: pip install -e .
      - name: Run full test suite
        run: pytest -m "not performance" -v
```

### Strategy 5: Nightly Build

**Goal**: Run everything including performance tests.

**Marker expression**:

```bash
# Run absolutely everything
pytest -v
```

**Expected runtime**: Unlimited (could be hours for performance tests)

**GitHub Actions configuration**:

```yaml
# .github/workflows/nightly.yml
name: Nightly Build

on:
  schedule:
    - cron: '0 2 * * *'  # Run at 2 AM UTC every day

jobs:
  full-suite-with-performance:
    runs-on: ubuntu-latest
    timeout-minutes: 120  # 2 hour timeout
    
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: |
          pip install -e .
          pip install pytest pytest-benchmark
      - name: Run all tests including performance
        run: pytest -v --benchmark-only
      - name: Generate test report
        if: always()
        run: |
          pytest --html=report.html --self-contained-html
      - name: Upload test report
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: test-report
          path: report.html
```

## Iteration 4: Documentation and Onboarding

### The Scenario

A new developer joins your team. They need to understand:

- How tests are organized
- Which markers to use
- How to run tests locally
- How CI works

### Solution: Comprehensive Testing Documentation

Create a testing guide:

```markdown
# Testing Guide

## Test Organization

Our test suite is organized by test type and feature:

```
tests/
├── unit/           # Fast, isolated tests (< 100ms)
├── integration/    # Tests with database/API (1-5s)
├── e2e/           # Full workflow tests (5-30s)
└── performance/   # Benchmarks and load tests (30s+)
```

## Marker Taxonomy

### Test Types (Required - Choose One)

- `@pytest.mark.unit`: Fast unit tests, no external dependencies
- `@pytest.mark.integration`: Integration tests with database/APIs
- `@pytest.mark.e2e`: End-to-end workflow tests

### Speed (Optional)

- `@pytest.mark.fast`: Tests under 100ms (for TDD workflow)
- `@pytest.mark.slow`: Tests over 1 second

### Categories (Optional - Multiple Allowed)

- `@pytest.mark.security`: Security-related tests
- `@pytest.mark.performance`: Performance benchmarks
- `@pytest.mark.smoke`: Critical path tests

### Requirements (Optional - Multiple Allowed)

- `@pytest.mark.requires_database`: Needs database connection
- `@pytest.mark.requires_network`: Needs network access
- `@pytest.mark.requires_auth`: Needs authentication

## Running Tests

### During Development (TDD)

```bash
# Fast feedback (< 1 second)
pytest -m "unit and fast"

# Or use the alias
pt
```

### Before Committing

```bash
# Run unit and integration tests (5-10 seconds)
pytest -m "(unit or integration) and not slow"

# Or use the Makefile
make test-commit
```

### Running Specific Test Types

```bash
# Unit tests only
pytest -m unit

# Integration tests only
pytest -m integration

# Security tests only
pytest -m security

# Everything except performance
pytest -m "not performance"
```

### Running Tests for Specific Features

```bash
# All tests for payment processing
pytest tests/unit/test_processing.py tests/integration/test_payment_flow.py

# Or use keyword filter
pytest -k payment
```

## Writing New Tests

### 1. Choose the Right Test Type

**Unit Test** if:
- Testing a single function/method
- No external dependencies
- Should run in < 100ms

**Integration Test** if:
- Testing component interactions
- Uses database or external APIs
- Takes 1-5 seconds

**E2E Test** if:
- Testing complete user workflow
- Involves multiple components
- Takes 5+ seconds

### 2. Place Test in Correct Directory

```python
# Unit test → tests/unit/test_feature.py
# Integration test → tests/integration/test_feature.py
# E2E test → tests/e2e/test_feature.py
```

### 3. Apply Correct Markers

```python
# Unit test template
@pytest.mark.unit
@pytest.mark.fast
def test_something():
    pass

# Integration test template
@pytest.mark.integration
@pytest.mark.slow
@pytest.mark.requires_database
def test_something(test_database):
    pass

# E2E test template
@pytest.mark.e2e
@pytest.mark.slow
@pytest.mark.requires_database
@pytest.mark.requires_network
def test_something(test_database):
    pass
```

### 4. Follow Naming Conventions

- Test files: `test_*.py`
- Test functions: `test_*`
- Test names should describe what's being tested:
  - ✅ `test_validate_credit_card_rejects_invalid_checksum`
  - ❌ `test_validation_1`

## CI/CD Pipeline

### Pull Requests

- Runs all tests except performance tests
- Must pass before merging
- Runtime: ~5 minutes

### Main Branch

- Runs smoke tests first (fail fast)
- Then runs full suite except performance
- Runtime: ~15 minutes

### Nightly Builds

- Runs everything including performance tests
- Generates detailed test reports
- Runtime: ~2 hours

## Troubleshooting

### Tests Fail Locally But Pass in CI

1. Check if you have all dependencies installed
2. Verify database is running (for integration tests)
3. Check environment variables

### Tests Are Slow

1. Check if test has correct markers (should it be marked `slow`?)
2. Consider if test should be unit test instead of integration
3. Look for unnecessary database operations

### Test Organization Violations

If you see errors like "Unit test has dependency markers", it means:

1. Test is marked as `unit` but uses external dependencies
2. Either change test to `integration` or remove dependencies
3. See conftest.py for full validation rules

## Getting Help

- Read this guide first
- Check existing tests for examples
- Ask in #testing Slack channel
- Review test organization standards in conftest.py
```

## Iteration 5: Measuring and Improving Test Organization

### The Scenario

You want to track test organization metrics:

- How many tests of each type?
- What's the average test runtime by type?
- Are tests properly categorized?
- Which tests are slowest?

### Solution: Test Metrics Collection

Create a script to analyze test organization:

```python
# scripts/analyze_tests.py
"""
Analyze test suite organization and generate metrics.

Usage:
    python scripts/analyze_tests.py
"""
import subprocess
import json
from collections import defaultdict
from pathlib import Path

def collect_test_info():
    """Collect information about all tests using pytest."""
    # Run pytest with JSON report
    result = subprocess.run(
        ["pytest", "--collect-only", "-q", "--json-report", "--json-report-file=test_report.json"],
        capture_output=True,
        text=True
    )
    
    # Parse JSON report
    with open("test_report.json") as f:
        report = json.load(f)
    
    return report["tests"]

def analyze_markers(tests):
    """Analyze marker usage across tests."""
    marker_counts = defaultdict(int)
    test_type_counts = defaultdict(int)
    tests_by_type = defaultdict(list)
    
    for test in tests:
        markers = test.get("markers", [])
        
        # Count all markers
        for marker in markers:
            marker_counts[marker] += 1
        
        # Count test types
        test_types = {"unit", "integration", "e2e"} & set(markers)
        if len(test_types) == 1:
            test_type = list(test_types)[0]
            test_type_counts[test_type] += 1
            tests_by_type[test_type].append(test["nodeid"])
        elif len(test_types) == 0:
            test_type_counts["untyped"] += 1
        else:
            test_type_counts["multiple_types"] += 1
    
    return {
        "marker_counts": dict(marker_counts),
        "test_type_counts": dict(test_type_counts),
        "tests_by_type": dict(tests_by_type)
    }

def analyze_directory_structure(tests):
    """Analyze test distribution across directories."""
    dir_counts = defaultdict(int)
    
    for test in tests:
        test_path = Path(test["nodeid"].split("::")[0])
        directory = test_path.parent
        dir_counts[str(directory)] += 1
    
    return dict(dir_counts)

def generate_report(analysis):
    """Generate human-readable report."""
    print("=" * 60)
    print("TEST SUITE ORGANIZATION REPORT")
    print("=" * 60)
    
    print("\n## Test Type Distribution")
    print("-" * 60)
    test_type_counts = analysis["test_type_counts"]
    total_tests = sum(test_type_counts.values())
    
    for test_type, count in sorted(test_type_counts.items()):
        percentage = (count / total_tests) * 100
        print(f"{test_type:20s}: {count:4d} ({percentage:5.1f}%)")
    
    print(f"\n{'Total':20s}: {total_tests:4d}")
    
    print("\n## Marker Usage")
    print("-" * 60)
    marker_counts = analysis["marker_counts"]
    
    for marker, count in sorted(marker_counts.items(), key=lambda x: x[1], reverse=True):
        print(f"{marker:30s}: {count:4d}")
    
    print("\n## Directory Distribution")
    print("-" * 60)
    dir_counts = analysis["directory_distribution"]
    
    for directory, count in sorted(dir_counts.items()):
        print(f"{directory:40s}: {count:4d}")
    
    # Warnings
    print("\n## Warnings")
    print("-" * 60)
    
    if test_type_counts.get("untyped", 0) > 0:
        print(f"⚠️  {test_type_counts['untyped']} tests missing test type marker")
    
    if test_type_counts.get("multiple_types", 0) > 0:
        print(f"⚠️  {test_type_counts['multiple_types']} tests have multiple type markers")
    
    # Calculate test pyramid ratio
    unit_count = test_type_counts.get("unit", 0)
    integration_count = test_type_counts.get("integration", 0)
    e2e_count = test_type_counts.get("e2e", 0)
    
    if e2e_count > 0:
        pyramid_ratio = unit_count / e2e_count
        print(f"\n📊 Test Pyramid Ratio (unit:e2e): {pyramid_ratio:.1f}:1")
        
        if pyramid_ratio < 5:
            print("   ⚠️  Consider adding more unit tests (recommended ratio: 10:1)")
        elif pyramid_ratio > 20:
            print("   ✅ Good test pyramid shape")

if __name__ == "__main__":
    print("Collecting test information...")
    tests = collect_test_info()
    
    print("Analyzing markers...")
    marker_analysis = analyze_markers(tests)
    
    print("Analyzing directory structure...")
    dir_analysis = analyze_directory_structure(tests)
    
    analysis = {
        **marker_analysis,
        "directory_distribution": dir_analysis
    }
    
    generate_report(analysis)
```

**Run the analysis**:

```bash
$ python scripts/analyze_tests.py
Collecting test information...
Analyzing markers...
Analyzing directory structure...
============================================================
TEST SUITE ORGANIZATION REPORT
============================================================

## Test Type Distribution
------------------------------------------------------------
unit                :  320 ( 64.0%)
integration         :  150 ( 30.0%)
e2e                 :   25 (  5.0%)
untyped             :    5 (  1.0%)

Total               :  500

## Marker Usage
------------------------------------------------------------
unit                              :  320
integration                       :  150
fast                              :  280
slow                              :  120
requires_database                 :  175
security                          :   45
smoke                             :   15
e2e                               :   25
requires_network                  :   30
performance                       :   10

## Directory Distribution
------------------------------------------------------------
tests/unit                        :  320
tests/integration                 :  150
tests/e2e                         :   25
tests/performance                 :   10

## Warnings
------------------------------------------------------------
⚠️  5 tests missing test type marker

📊 Test Pyramid Ratio (unit:e2e): 12.8:1
   ✅ Good test pyramid shape
```

**What this reveals**:

1. **Test distribution**: 64% unit, 30% integration, 5% e2e (healthy pyramid)
2. **Marker usage**: Most common markers and their frequency
3. **Directory alignment**: Tests are in correct directories
4. **Issues**: 5 tests need type markers
5. **Test pyramid**: Good ratio of unit to e2e tests

## Best Practices Summary

### 1. Establish Clear Categories

**Do**:
- Define 2-3 primary dimensions (type, speed, category)
- Make test types mutually exclusive
- Document what each marker means

**Don't**:
- Create too many markers (causes confusion)
- Use ambiguous marker names
- Let markers overlap in meaning

### 2. Align Structure with Markers

**Do**:
- Organize files by test type (unit/, integration/, e2e/)
- Use markers to add cross-cutting concerns (security, performance)
- Keep related tests together

**Don't**:
- Put unit tests in integration/ directory
- Scatter related tests across many files
- Rely solely on markers for organization

### 3. Enforce Standards

**Do**:
- Use pytest hooks to validate marker usage
- Fail CI if standards violated
- Document standards clearly

**Don't**:
- Let standards drift over time
- Make standards too strict (blocks productivity)
- Enforce without documentation

### 4. Optimize for Common Workflows

**Do**:
- Create aliases for common test runs
- Configure CI for different contexts
- Measure and optimize slow tests

**Don't**:
- Run full suite for every change
- Ignore slow tests (they accumulate)
- Make developers wait for feedback

### 5. Measure and Improve

**Do**:
- Track test metrics over time
- Review test organization regularly
- Refactor tests as project evolves

**Don't**:
- Let test suite grow without review
- Ignore test pyramid violations
- Keep obsolete tests

## What You've Learned

Marker-based test organization provides a scalable framework for managing large test suites:

1. **Test taxonomy**: Define clear categories (type, speed, category, requirements)
2. **File structure**: Organize by test type, use markers for cross-cutting concerns
3. **Enforcement**: Use pytest hooks to validate standards
4. **Execution strategies**: Different contexts need different test subsets
5. **Documentation**: Comprehensive guides help team members
6. **Metrics**: Track organization health over time

**Key insights**:

- Good organization scales from 10 to 10,000 tests
- Markers complement (don't replace) file structure
- Enforcement prevents drift over time
- Different contexts need different test execution strategies
- Measurement reveals organization issues

**The journey complete**: You've learned to use markers to organize tests from simple categorization to comprehensive test suite management. This foundation enables efficient testing at any scale.
