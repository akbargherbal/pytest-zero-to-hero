# Chapter 17: Debugging Failing Tests

## Reading Test Output

## The Anatomy of a Test Failure

When a test fails, pytest doesn't just tell you "something broke"—it provides a detailed narrative of exactly what happened, where it happened, and why it happened. Learning to read this output systematically transforms debugging from guesswork into detective work.

Let's establish our reference implementation: a payment processing system that we'll use throughout this chapter to demonstrate debugging techniques.

```python
# payment_processor.py
class PaymentProcessor:
    def __init__(self, api_client):
        self.api_client = api_client
        self.transaction_log = []
    
    def process_payment(self, amount, card_number, cvv):
        """Process a payment and return transaction ID."""
        if amount <= 0:
            raise ValueError("Amount must be positive")
        
        if len(card_number) != 16:
            raise ValueError("Invalid card number")
        
        if len(str(cvv)) != 3:
            raise ValueError("Invalid CVV")
        
        # Simulate API call
        response = self.api_client.charge(
            amount=amount,
            card=card_number,
            cvv=cvv
        )
        
        transaction_id = response['transaction_id']
        self.transaction_log.append({
            'id': transaction_id,
            'amount': amount,
            'status': 'completed'
        })
        
        return transaction_id
    
    def get_transaction(self, transaction_id):
        """Retrieve a transaction by ID."""
        for transaction in self.transaction_log:
            if transaction['id'] == transaction_id:
                return transaction
        return None

class MockAPIClient:
    def __init__(self, should_fail=False):
        self.should_fail = should_fail
        self.call_count = 0
    
    def charge(self, amount, card, cvv):
        self.call_count += 1
        if self.should_fail:
            raise ConnectionError("API unavailable")
        return {
            'transaction_id': f'TXN_{self.call_count}',
            'status': 'success'
        }
```

```python
# test_payment_processor.py
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_successful_payment():
    """Test that a valid payment is processed correctly."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    assert transaction_id == 'TXN_1'
    assert api_client.call_count == 1
    
    # Verify transaction was logged
    transaction = processor.get_transaction(transaction_id)
    assert transaction['amount'] == 100.00
    assert transaction['status'] == 'completed'
```

This test passes. Now let's introduce a realistic bug and examine the failure output in detail.

```python
# test_payment_processor.py (continued)
def test_payment_with_decimal_amount():
    """Test payment with precise decimal amount."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    transaction_id = processor.process_payment(
        amount=99.99,
        card_number='1234567890123456',
        cvv='123'
    )
    
    transaction = processor.get_transaction(transaction_id)
    # Bug: We expect exact decimal match, but floating point arithmetic might fail
    assert transaction['amount'] == 99.99
```

Let's run this test and capture the complete output:

```bash
$ pytest test_payment_processor.py::test_payment_with_decimal_amount -v
```

**The complete output**:

```
============================= test session starts ==============================
platform linux -- Python 3.11.0, pytest-7.4.3, pluggy-1.3.0 -- /usr/bin/python3
cachedir: .pytest_cache
rootdir: /home/user/project
collected 1 item

test_payment_processor.py::test_payment_with_decimal_amount PASSED      [100%]

============================== 1 passed in 0.02s ===============================
```

Wait—this test passed! That's because our simple implementation doesn't have the floating-point bug yet. Let's introduce a more realistic scenario where the bug manifests.

```python
# payment_processor.py (modified)
class PaymentProcessor:
    def __init__(self, api_client):
        self.api_client = api_client
        self.transaction_log = []
        self.fee_percentage = 0.029  # 2.9% processing fee
    
    def process_payment(self, amount, card_number, cvv):
        """Process a payment and return transaction ID."""
        if amount <= 0:
            raise ValueError("Amount must be positive")
        
        if len(card_number) != 16:
            raise ValueError("Invalid card number")
        
        if len(str(cvv)) != 3:
            raise ValueError("Invalid CVV")
        
        # Calculate fee and net amount
        fee = amount * self.fee_percentage
        net_amount = amount - fee
        
        response = self.api_client.charge(
            amount=amount,
            card=card_number,
            cvv=cvv
        )
        
        transaction_id = response['transaction_id']
        self.transaction_log.append({
            'id': transaction_id,
            'amount': amount,
            'fee': fee,
            'net_amount': net_amount,
            'status': 'completed'
        })
        
        return transaction_id
    
    def get_transaction(self, transaction_id):
        """Retrieve a transaction by ID."""
        for transaction in self.transaction_log:
            if transaction['id'] == transaction_id:
                return transaction
        return None
```

```python
# test_payment_processor.py (updated test)
def test_payment_fee_calculation():
    """Test that processing fees are calculated correctly."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    transaction = processor.get_transaction(transaction_id)
    
    # Expected: 2.9% of $100.00 = $2.90
    assert transaction['fee'] == 2.90
    # Expected: $100.00 - $2.90 = $97.10
    assert transaction['net_amount'] == 97.10
```

Now let's run this test:

```bash
$ pytest test_payment_processor.py::test_payment_fee_calculation -v
```

**The complete output**:

```
============================= test session starts ==============================
platform linux -- Python 3.11.0, pytest-7.4.3, pluggy-1.3.0 -- /usr/bin/python3
cachedir: .pytest_cache
rootdir: /home/user/project
collected 1 item

test_payment_processor.py::test_payment_fee_calculation FAILED          [100%]

=================================== FAILURES ===================================
___________________________ test_payment_fee_calculation _______________________

    def test_payment_fee_calculation():
        """Test that processing fees are calculated correctly."""
        api_client = MockAPIClient()
        processor = PaymentProcessor(api_client)
        
        transaction_id = processor.process_payment(
            amount=100.00,
            card_number='1234567890123456',
            cvv='123'
        )
        
        transaction = processor.get_transaction(transaction_id)
        
        # Expected: 2.9% of $100.00 = $2.90
>       assert transaction['fee'] == 2.90
E       AssertionError: assert 2.8999999999999995 == 2.90
E        +  where 2.8999999999999995 = {'id': 'TXN_1', 'amount': 100.0, 'fee': 2.8999999999999995, 'net_amount': 97.10000000000001, 'status': 'completed'}['fee']

test_payment_processor.py:25: AssertionError
=========================== 1 passed, 1 failed in 0.03s =======================
```

Perfect! Now we have a real failure to dissect.

### Diagnostic Analysis: Reading the Failure

Let's parse this output section by section to extract every piece of information pytest provides.

**Section 1: The Test Session Header**

```
============================= test session starts ==============================
platform linux -- Python 3.11.0, pytest-7.4.3, pluggy-1.3.0 -- /usr/bin/python3
cachedir: .pytest_cache
rootdir: /home/user/project
collected 1 item
```

**What this tells us**:
- **Platform and versions**: We're running on Linux with Python 3.11.0 and pytest 7.4.3
- **Cache location**: Pytest stores test results in `.pytest_cache` for features like `--lf` (last failed)
- **Root directory**: The project root is `/home/user/project`
- **Collection count**: Pytest found exactly 1 test to run

**Why this matters**: Version information helps reproduce issues. If a test passes on your machine but fails in CI, version differences are the first suspect.

**Section 2: The Test Execution Line**

```
test_payment_processor.py::test_payment_fee_calculation FAILED          [100%]
```

**What this tells us**:
- **File location**: `test_payment_processor.py`
- **Test name**: `test_payment_fee_calculation`
- **Result**: `FAILED`
- **Progress**: `[100%]` means this was the only test (or the last of multiple tests)

**Why this matters**: The `::` syntax shows you exactly how to re-run just this test: `pytest test_payment_processor.py::test_payment_fee_calculation`

**Section 3: The Failures Section Header**

```
=================================== FAILURES ===================================
___________________________ test_payment_fee_calculation _______________________
```

**What this tells us**:
- This section contains detailed information about all failed tests
- The underscored line shows which specific test failed

**Section 4: The Traceback**

```
    def test_payment_fee_calculation():
        """Test that processing fees are calculated correctly."""
        api_client = MockAPIClient()
        processor = PaymentProcessor(api_client)
        
        transaction_id = processor.process_payment(
            amount=100.00,
            card_number='1234567890123456',
            cvv='123'
        )
        
        transaction = processor.get_transaction(transaction_id)
        
        # Expected: 2.9% of $100.00 = $2.90
>       assert transaction['fee'] == 2.90
```

**What this tells us**:
- The `>` marker points to the exact line that failed
- We can see the full context of the test function
- The failure occurred at line 25 in the test file

**Why this matters**: The traceback shows you the execution path. For complex tests with multiple assertions, this pinpoints exactly which assertion failed.

**Section 5: The Assertion Introspection**

```
E       AssertionError: assert 2.8999999999999995 == 2.90
E        +  where 2.8999999999999995 = {'id': 'TXN_1', 'amount': 100.0, 'fee': 2.8999999999999995, 'net_amount': 97.10000000000001, 'status': 'completed'}['fee']
```

**What this tells us**:
- **The comparison**: `2.8999999999999995 == 2.90` (False)
- **The actual value**: `2.8999999999999995`
- **The expected value**: `2.90`
- **The source**: The value came from `transaction['fee']`
- **The full object**: We can see the entire transaction dictionary

**Why this matters**: This is pytest's "assertion introspection"—it shows you not just that the assertion failed, but the actual values involved and where they came from. Notice how pytest automatically expanded `transaction['fee']` to show us the full dictionary.

**Section 6: The Location Reference**

```
test_payment_processor.py:25: AssertionError
```

**What this tells us**:
- The exact file and line number where the assertion failed
- The exception type: `AssertionError`

**Section 7: The Summary Line**

```
=========================== 1 passed, 1 failed in 0.03s =======================
```

**What this tells us**:
- **Total results**: 1 test passed (from earlier), 1 test failed
- **Execution time**: 0.03 seconds
- **Overall status**: The test suite failed

### Root Cause Identified

The failure reveals a **floating-point precision issue**. When we calculate `100.00 * 0.029`, Python's floating-point arithmetic produces `2.8999999999999995` instead of the exact decimal `2.90`.

### Why the Current Approach Can't Solve This

Using exact equality (`==`) for floating-point comparisons is fundamentally flawed because:
1. Binary floating-point cannot represent most decimal fractions exactly
2. Arithmetic operations accumulate rounding errors
3. The error is mathematically unavoidable in IEEE 754 floating-point

### What We Need

A comparison method that allows for acceptable tolerance in floating-point assertions. This is where `pytest.approx()` becomes essential.

## Reading Output Systematically: A Checklist

When a test fails, read the output in this order:

### 1. Start with the Summary Line (Bottom)
- How many tests failed?
- How many passed?
- Is this an isolated failure or part of a larger pattern?

### 2. Identify the Failed Test (Failures Section)
- Which test failed?
- What file is it in?
- Can you run just this test in isolation?

### 3. Locate the Failure Point (Traceback)
- Which line has the `>` marker?
- What assertion failed?
- What was the test trying to verify?

### 4. Understand the Values (Assertion Introspection)
- What was the actual value?
- What was the expected value?
- How do they differ?
- Can you see the full objects involved?

### 5. Check the Context (Full Traceback)
- What code ran before the failure?
- Are there any clues in the setup?
- Did the test get as far as you expected?

### 6. Look for Patterns (Multiple Failures)
- If multiple tests failed, do they share common characteristics?
- Are they all in the same module?
- Do they all test the same functionality?

## Common Output Patterns and What They Mean

### Pattern 1: AssertionError with Value Comparison

```
E       AssertionError: assert 2.8999999999999995 == 2.90
```

**Diagnosis**: The assertion compared two values and they didn't match.
**Common causes**: 
- Floating-point precision issues
- Off-by-one errors
- Incorrect expected values
- Logic bugs in the code under test

### Pattern 2: AttributeError

```
E       AttributeError: 'NoneType' object has no attribute 'fee'
```

**Diagnosis**: You tried to access an attribute on `None`.
**Common causes**:
- A function returned `None` instead of an object
- A lookup failed (like `get_transaction()` returning `None`)
- Initialization didn't happen

### Pattern 3: KeyError

```
E       KeyError: 'transaction_id'
```

**Diagnosis**: You tried to access a dictionary key that doesn't exist.
**Common causes**:
- Typo in the key name
- The API response structure changed
- The dictionary wasn't populated as expected

### Pattern 4: TypeError

```
E       TypeError: unsupported operand type(s) for -: 'str' and 'float'
```

**Diagnosis**: You tried to perform an operation on incompatible types.
**Common causes**:
- Data wasn't converted to the expected type
- API returned strings instead of numbers
- Missing type validation

### Pattern 5: IndexError

```
E       IndexError: list index out of range
```

**Diagnosis**: You tried to access a list element that doesn't exist.
**Common causes**:
- Empty list when you expected items
- Off-by-one error in indexing
- Loop logic error

## The Power of Assertion Introspection

Pytest's assertion introspection is one of its most powerful features. Let's see it in action with different types of comparisons.

```python
# test_introspection_examples.py
def test_list_comparison():
    """Demonstrate introspection for list comparison."""
    expected = [1, 2, 3, 4, 5]
    actual = [1, 2, 3, 4, 6]
    assert actual == expected

def test_dict_comparison():
    """Demonstrate introspection for dictionary comparison."""
    expected = {
        'name': 'Alice',
        'age': 30,
        'email': 'alice@example.com'
    }
    actual = {
        'name': 'Alice',
        'age': 31,
        'email': 'alice@example.com'
    }
    assert actual == expected

def test_string_comparison():
    """Demonstrate introspection for string comparison."""
    expected = "The quick brown fox jumps over the lazy dog"
    actual = "The quick brown fox jumps over the lazy cat"
    assert actual == expected
```

Running these tests produces detailed output showing exactly where the differences are:

```bash
$ pytest test_introspection_examples.py -v
```

**Output for list comparison**:

```
E       AssertionError: assert [1, 2, 3, 4, 6] == [1, 2, 3, 4, 5]
E         At index 4 diff: 6 != 5
E         Use -v to get more diff
```

**What this tells us**: Pytest identified that the lists differ at index 4, where we have `6` instead of `5`.

**Output for dictionary comparison**:

```
E       AssertionError: assert {'age': 31, '...e@example.com'} == {'age': 30, '...e@example.com'}
E         Omitting 2 identical items, use -vv to show
E         Differing items:
E         {'age': 31} != {'age': 30}
```

**What this tells us**: Pytest shows only the differing keys, omitting the identical ones. The `age` field is `31` but should be `30`.

**Output for string comparison**:

```
E       AssertionError: assert 'The quick br...r the lazy cat' == 'The quick br...r the lazy dog'
E         - The quick brown fox jumps over the lazy dog
E         ?                                         ^^^
E         + The quick brown fox jumps over the lazy cat
E         ?                                         ^^^
```

**What this tells us**: Pytest shows a diff-style comparison with `?` markers pointing to the exact characters that differ.

## Practical Exercise: Reading Real Failure Output

Let's create a more complex scenario that demonstrates multiple types of failures.

```python
# test_complex_scenario.py
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_multiple_payments_tracking():
    """Test that multiple payments are tracked correctly."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    # Process three payments
    txn1 = processor.process_payment(100.00, '1234567890123456', '123')
    txn2 = processor.process_payment(200.00, '1234567890123456', '123')
    txn3 = processor.process_payment(300.00, '1234567890123456', '123')
    
    # Verify all transactions are logged
    assert len(processor.transaction_log) == 3
    
    # Verify transaction IDs are unique
    transaction_ids = [txn1, txn2, txn3]
    assert len(set(transaction_ids)) == 3
    
    # Verify we can retrieve each transaction
    for txn_id in transaction_ids:
        transaction = processor.get_transaction(txn_id)
        assert transaction is not None
        assert transaction['status'] == 'completed'
    
    # Verify total amount processed
    total = sum(t['amount'] for t in processor.transaction_log)
    assert total == 600.00
    
    # Verify API was called correct number of times
    assert api_client.call_count == 3
```

This test passes. Now let's introduce a bug in the `get_transaction` method:

```python
# payment_processor.py (with bug)
def get_transaction(self, transaction_id):
    """Retrieve a transaction by ID."""
    # Bug: Using 'is' instead of '==' for string comparison
    for transaction in self.transaction_log:
        if transaction['id'] is transaction_id:
            return transaction
    return None
```

Now when we run the test:

```bash
$ pytest test_complex_scenario.py::test_multiple_payments_tracking -v
```

**The output**:

```
=================================== FAILURES ===================================
________________________ test_multiple_payments_tracking _______________________

    def test_multiple_payments_tracking():
        """Test that multiple payments are tracked correctly."""
        api_client = MockAPIClient()
        processor = PaymentProcessor(api_client)
        
        # Process three payments
        txn1 = processor.process_payment(100.00, '1234567890123456', '123')
        txn2 = processor.process_payment(200.00, '1234567890123456', '123')
        txn3 = processor.process_payment(300.00, '1234567890123456', '123')
        
        # Verify all transactions are logged
        assert len(processor.transaction_log) == 3
        
        # Verify transaction IDs are unique
        transaction_ids = [txn1, txn2, txn3]
        assert len(set(transaction_ids)) == 3
        
        # Verify we can retrieve each transaction
        for txn_id in transaction_ids:
            transaction = processor.get_transaction(txn_id)
>           assert transaction is not None
E           AssertionError: assert None is not None

test_complex_scenario.py:20: AssertionError
```

### Diagnostic Analysis: The Identity Comparison Bug

**The failure point**: `assert transaction is not None`

**What this tells us**:
- `get_transaction()` returned `None`
- This happened inside a loop, so at least one transaction lookup failed
- The transaction IDs exist (we just created them), so the lookup logic must be broken

**The root cause**: Using `is` for string comparison instead of `==`. The `is` operator checks object identity (same memory location), not value equality. String literals might be interned, but dynamically created strings won't be.

**The fix**:

```python
# payment_processor.py (fixed)
def get_transaction(self, transaction_id):
    """Retrieve a transaction by ID."""
    for transaction in self.transaction_log:
        if transaction['id'] == transaction_id:  # Use == for value comparison
            return transaction
    return None
```

## Key Takeaways: Reading Test Output

### 1. Output is Structured Information, Not Noise
Every section of pytest's output serves a specific purpose. Learn to scan for the information you need.

### 2. Start from the Bottom, Work Up
The summary line tells you the scope of the problem. The traceback tells you the location. The introspection tells you the cause.

### 3. Assertion Introspection is Your Best Friend
Pytest automatically shows you the values involved in failed assertions. Use this to understand what went wrong without adding print statements.

### 4. The `>` Marker is Your Target
This shows you exactly which line failed. Everything else is context.

### 5. Read the Full Object, Not Just the Comparison
When pytest shows `where X = {...}`, read the entire object. Often the bug is in a related field, not the one you're comparing.

### 6. Version Information Matters
The test session header shows Python and pytest versions. When debugging environment-specific failures, this is your first clue.

### 7. File and Line Numbers are Clickable
Most IDEs and terminals make file paths clickable. Use this to jump directly to the failure point.

In the next section, we'll explore how pytest's verbose modes can provide even more detailed information when you need it.

## Using pytest's Verbose and Extra-Verbose Modes

## Controlling Output Detail

Pytest provides multiple verbosity levels to control how much information you see. By default, pytest shows a minimal summary. When debugging, you often need more detail.

### The Verbosity Spectrum

- **Default mode** (`pytest`): Minimal output, dots for passing tests
- **Verbose mode** (`pytest -v`): Show test names and outcomes
- **Extra-verbose mode** (`pytest -vv`): Show detailed assertion comparisons
- **Super-verbose mode** (`pytest -vvv`): Show even more internal details

Let's see each mode in action using our payment processor.

```python
# test_verbosity_demo.py
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_payment_validation_positive_amount():
    """Test that positive amounts are accepted."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    transaction_id = processor.process_payment(
        amount=50.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    assert transaction_id is not None

def test_payment_validation_zero_amount():
    """Test that zero amount is rejected."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    with pytest.raises(ValueError, match="Amount must be positive"):
        processor.process_payment(
            amount=0.00,
            card_number='1234567890123456',
            cvv='123'
        )

def test_payment_validation_negative_amount():
    """Test that negative amount is rejected."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    with pytest.raises(ValueError, match="Amount must be positive"):
        processor.process_payment(
            amount=-10.00,
            card_number='1234567890123456',
            cvv='123'
        )

def test_payment_validation_invalid_card():
    """Test that invalid card number is rejected."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    with pytest.raises(ValueError, match="Invalid card number"):
        processor.process_payment(
            amount=50.00,
            card_number='12345',  # Too short
            cvv='123'
        )

def test_payment_validation_invalid_cvv():
    """Test that invalid CVV is rejected."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    with pytest.raises(ValueError, match="Invalid CVV"):
        processor.process_payment(
            amount=50.00,
            card_number='1234567890123456',
            cvv='12'  # Too short
        )
```

### Default Mode: Minimal Output

```bash
$ pytest test_verbosity_demo.py
```

**Output**:

```
============================= test session starts ==============================
platform linux -- Python 3.11.0, pytest-7.4.3, pluggy-1.3.0
rootdir: /home/user/project
collected 5 items

test_verbosity_demo.py .....                                             [100%]

============================== 5 passed in 0.02s ===============================
```

**What you see**:
- Five dots (`.....`) representing five passing tests
- Total count and execution time
- No test names, no details

**When to use**: Quick smoke tests, CI pipelines where you only care about pass/fail.

### Verbose Mode (-v): Show Test Names

```bash
$ pytest test_verbosity_demo.py -v
```

**Output**:

```
============================= test session starts ==============================
platform linux -- Python 3.11.0, pytest-7.4.3, pluggy-1.3.0 -- /usr/bin/python3
cachedir: .pytest_cache
rootdir: /home/user/project
collected 5 items

test_verbosity_demo.py::test_payment_validation_positive_amount PASSED   [ 20%]
test_verbosity_demo.py::test_payment_validation_zero_amount PASSED       [ 40%]
test_verbosity_demo.py::test_payment_validation_negative_amount PASSED   [ 60%]
test_verbosity_demo.py::test_payment_validation_invalid_card PASSED      [ 80%]
test_verbosity_demo.py::test_payment_validation_invalid_cvv PASSED       [100%]

============================== 5 passed in 0.02s ===============================
```

**What you see**:
- Full test names with file paths
- Individual pass/fail status for each test
- Progress percentage
- Execution order

**When to use**: 
- Debugging to see which specific test failed
- Verifying test discovery found the right tests
- Monitoring test execution order
- Most development work

### Extra-Verbose Mode (-vv): Detailed Comparisons

Now let's introduce a failing test to see the difference `-vv` makes:

```python
# test_verbosity_demo.py (add this test)
def test_transaction_details_match():
    """Test that transaction details are recorded accurately."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    transaction_id = processor.process_payment(
        amount=150.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    transaction = processor.get_transaction(transaction_id)
    
    expected = {
        'id': 'TXN_1',
        'amount': 150.00,
        'fee': 4.35,  # 2.9% of 150.00
        'net_amount': 145.65,
        'status': 'completed'
    }
    
    assert transaction == expected
```

First, run with `-v`:

```bash
$ pytest test_verbosity_demo.py::test_transaction_details_match -v
```

**Output with -v**:

```
=================================== FAILURES ===================================
_________________________ test_transaction_details_match _______________________

    def test_transaction_details_match():
        """Test that transaction details are recorded accurately."""
        api_client = MockAPIClient()
        processor = PaymentProcessor(api_client)
        
        transaction_id = processor.process_payment(
            amount=150.00,
            card_number='1234567890123456',
            cvv='123'
        )
        
        transaction = processor.get_transaction(transaction_id)
        
        expected = {
            'id': 'TXN_1',
            'amount': 150.00,
            'fee': 4.35,  # 2.9% of 150.00
            'net_amount': 145.65,
            'status': 'completed'
        }
        
>       assert transaction == expected
E       AssertionError: assert {'amount': 15...completed'} == {'amount': 15...completed'}
E         Omitting 3 identical items, use -vv to show
E         Differing items:
E         {'fee': 4.3499999999999996} != {'fee': 4.35}
E         {'net_amount': 145.65000000000003} != {'net_amount': 145.65}

test_verbosity_demo.py:25: AssertionError
```

**What you see**:
- Pytest tells you there are 3 identical items but doesn't show them
- Shows only the differing items
- Suggests using `-vv` for more detail

Now run with `-vv`:

```bash
$ pytest test_verbosity_demo.py::test_transaction_details_match -vv
```

**Output with -vv**:

```
=================================== FAILURES ===================================
_________________________ test_transaction_details_match _______________________

    def test_transaction_details_match():
        """Test that transaction details are recorded accurately."""
        api_client = MockAPIClient()
        processor = PaymentProcessor(api_client)
        
        transaction_id = processor.process_payment(
            amount=150.00,
            card_number='1234567890123456',
            cvv='123'
        )
        
        transaction = processor.get_transaction(transaction_id)
        
        expected = {
            'id': 'TXN_1',
            'amount': 150.00,
            'fee': 4.35,  # 2.9% of 150.00
            'net_amount': 145.65,
            'status': 'completed'
        }
        
>       assert transaction == expected
E       AssertionError: assert {'amount': 150.0,
E        'fee': 4.3499999999999996,
E        'id': 'TXN_1',
E        'net_amount': 145.65000000000003,
E        'status': 'completed'} == {'amount': 150.0,
E        'fee': 4.35,
E        'id': 'TXN_1',
E        'net_amount': 145.65,
E        'status': 'completed'}
E       
E       Differing items:
E       {'fee': 4.3499999999999996} != {'fee': 4.35}
E       {'net_amount': 145.65000000000003} != {'net_amount': 145.65}

test_verbosity_demo.py:25: AssertionError
```

**What you see**:
- Full dictionary contents for both actual and expected
- Pretty-printed formatting showing structure
- All items, not just differing ones
- Easier to see the complete context

**When to use `-vv`**:
- Debugging complex data structure comparisons
- When you need to see all fields, not just differences
- Investigating why two "similar" objects aren't equal
- Understanding the full state of objects in assertions

### Super-Verbose Mode (-vvv): Internal Details

The `-vvv` flag shows pytest's internal operations:

```bash
$ pytest test_verbosity_demo.py -vvv
```

**Additional output with -vvv**:

```
============================= test session starts ==============================
platform linux -- Python 3.11.0, pytest-7.4.3, pluggy-1.3.0 -- /usr/bin/python3
cachedir: .pytest_cache
rootdir: /home/user/project
plugins: asyncio-0.21.0, cov-4.1.0
collected 6 items

test_verbosity_demo.py::test_payment_validation_positive_amount 
PASSED                                                                   [ 16%]
test_verbosity_demo.py::test_payment_validation_zero_amount 
PASSED                                                                   [ 33%]
test_verbosity_demo.py::test_payment_validation_negative_amount 
PASSED                                                                   [ 50%]
test_verbosity_demo.py::test_payment_validation_invalid_card 
PASSED                                                                   [ 66%]
test_verbosity_demo.py::test_payment_validation_invalid_cvv 
PASSED                                                                   [ 83%]
test_verbosity_demo.py::test_transaction_details_match 
FAILED                                                                   [100%]
```

**What you see**:
- Loaded plugins listed
- Each test on its own line
- More spacing for readability

**When to use `-vvv`**:
- Debugging pytest plugin issues
- Understanding test collection problems
- Investigating fixture execution order
- Rarely needed for normal debugging

## Combining Verbosity with Other Flags

Verbosity flags work well with other pytest options:

### Show Local Variables on Failure: `-l` or `--showlocals`

```bash
$ pytest test_verbosity_demo.py::test_transaction_details_match -vv -l
```

**Output**:

```
=================================== FAILURES ===================================
_________________________ test_transaction_details_match _______________________

    def test_transaction_details_match():
        """Test that transaction details are recorded accurately."""
        api_client = MockAPIClient()
        processor = PaymentProcessor(api_client)
        
        transaction_id = processor.process_payment(
            amount=150.00,
            card_number='1234567890123456',
            cvv='123'
        )
        
        transaction = processor.get_transaction(transaction_id)
        
        expected = {
            'id': 'TXN_1',
            'amount': 150.00,
            'fee': 4.35,
            'net_amount': 145.65,
            'status': 'completed'
        }
        
>       assert transaction == expected
E       AssertionError: assert {'amount': 150.0,
E        'fee': 4.3499999999999996,
E        'id': 'TXN_1',
E        'net_amount': 145.65000000000003,
E        'status': 'completed'} == {'amount': 150.0,
E        'fee': 4.35,
E        'id': 'TXN_1',
E        'net_amount': 145.65,
E        'status': 'completed'}
E       
E       Differing items:
E       {'fee': 4.3499999999999996} != {'fee': 4.35}
E       {'net_amount': 145.65000000000003} != {'net_amount': 145.65}

api_client = <payment_processor.MockAPIClient object at 0x7f8b3c4d5e90>
expected   = {'amount': 150.0, 'fee': 4.35, 'id': 'TXN_1', 'net_amount': 145.65, 'status': 'completed'}
processor  = <payment_processor.PaymentProcessor object at 0x7f8b3c4d5f10>
transaction = {'amount': 150.0, 'fee': 4.3499999999999996, 'id': 'TXN_1', 'net_amount': 145.65000000000003, ...}
transaction_id = 'TXN_1'

test_verbosity_demo.py:25: AssertionError
```

**What you see**:
- All local variables at the point of failure
- Their values and types
- Memory addresses for objects

**When to use**: When you need to see the state of all variables, not just those in the assertion.

### Show Full Traceback: `--tb=long`

```bash
$ pytest test_verbosity_demo.py::test_transaction_details_match -vv --tb=long
```

**What you see**:
- Complete traceback including all function calls
- Full source code for each frame
- More context about the execution path

**Traceback styles**:
- `--tb=auto` (default): Intelligent selection based on failure type
- `--tb=long`: Full traceback with source code
- `--tb=short`: Shorter traceback without source
- `--tb=line`: Single line per failure
- `--tb=native`: Python's standard traceback format
- `--tb=no`: No traceback at all

### Show Captured Output: `-s` or `--capture=no`

By default, pytest captures stdout and stderr. Use `-s` to see print statements in real-time:

```python
# test_verbosity_demo.py (add this test)
def test_with_debug_output():
    """Test with debug print statements."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    print("Starting payment processing...")
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    print(f"Transaction ID: {transaction_id}")
    
    transaction = processor.get_transaction(transaction_id)
    print(f"Transaction details: {transaction}")
    
    assert transaction['amount'] == 100.00
```

```bash
$ pytest test_verbosity_demo.py::test_with_debug_output -v -s
```

**Output**:

```
test_verbosity_demo.py::test_with_debug_output Starting payment processing...
Transaction ID: TXN_1
Transaction details: {'id': 'TXN_1', 'amount': 100.0, 'fee': 2.8999999999999995, 'net_amount': 97.10000000000001, 'status': 'completed'}
PASSED
```

**What you see**:
- Print statements appear in real-time
- Output is interleaved with test execution
- Useful for debugging test flow

## Practical Verbosity Strategies

### Strategy 1: Start Minimal, Increase as Needed

```bash
# First run: Quick check
$ pytest

# If failures: See which tests failed
$ pytest -v

# If complex data: See full comparisons
$ pytest -vv

# If still stuck: See local variables
$ pytest -vv -l

# If really stuck: See everything
$ pytest -vvv -l --tb=long -s
```

### Strategy 2: Use Configuration for Default Verbosity

In `pytest.ini`:

```ini
[pytest]
# Always use verbose mode
addopts = -v

# Or for extra detail
addopts = -vv --tb=short
```

### Strategy 3: Verbosity for Specific Test Runs

Use different verbosity for different scenarios:

```bash
# Quick smoke test (minimal output)
$ pytest -q

# Development (verbose)
$ pytest -v

# Debugging specific failure (maximum detail)
$ pytest test_file.py::test_name -vv -l --tb=long -s

# CI pipeline (verbose but compact)
$ pytest -v --tb=short
```

### Strategy 4: Quiet Mode for Clean Output

Use `-q` or `--quiet` for minimal output:

```bash
$ pytest -q
```

**Output**:

```
.....F                                                                   [100%]
=================================== FAILURES ===================================
[failure details]
1 failed, 5 passed in 0.02s
```

**When to use**: 
- CI pipelines where you only need failure details
- Running large test suites where dots are sufficient
- Scripts that parse pytest output

## Decision Framework: Which Verbosity Level?

| Scenario | Recommended Flags | Why |
|----------|------------------|-----|
| Quick smoke test | `pytest` or `pytest -q` | Minimal noise, just pass/fail |
| Development work | `pytest -v` | See test names, track progress |
| Debugging data structures | `pytest -vv` | Full object comparisons |
| Debugging test flow | `pytest -v -s` | See print statements |
| Debugging complex failure | `pytest -vv -l --tb=long` | Maximum context |
| CI pipeline | `pytest -v --tb=short` | Readable but compact |
| Plugin debugging | `pytest -vvv` | Internal pytest details |

## Common Verbosity Patterns

### Pattern 1: The Progressive Debug

Start minimal and add flags as you need more information:

```bash
# Step 1: See if it fails
$ pytest test_file.py

# Step 2: See which test failed
$ pytest test_file.py -v

# Step 3: See the full comparison
$ pytest test_file.py::test_name -vv

# Step 4: See local variables
$ pytest test_file.py::test_name -vv -l

# Step 5: See print statements
$ pytest test_file.py::test_name -vv -l -s
```

### Pattern 2: The Focused Investigation

When you know which test is failing:

```bash
# Run just that test with maximum detail
$ pytest test_file.py::test_name -vv -l --tb=long -s
```

### Pattern 3: The CI-Friendly Run

For continuous integration:

```bash
# Verbose enough to debug, compact enough to read
$ pytest -v --tb=short --maxfail=1
```

## Key Takeaways: Verbosity Modes

### 1. Verbosity is a Spectrum, Not a Binary Choice
You have fine-grained control over output detail. Use the minimum verbosity needed to solve your current problem.

### 2. `-vv` is Your Friend for Data Debugging
When assertions fail on complex data structures, `-vv` shows you the complete picture.

### 3. Combine Flags for Maximum Insight
`-vv -l --tb=long -s` gives you everything: full comparisons, local variables, complete tracebacks, and print output.

### 4. Configure Defaults for Your Workflow
Set `addopts = -v` in `pytest.ini` if you always want verbose output.

### 5. Different Scenarios Need Different Verbosity
Quick checks need minimal output. Deep debugging needs maximum detail. Choose appropriately.

### 6. Quiet Mode is Underrated
`-q` is perfect for scripts and CI where you only care about failures.

In the next section, we'll explore the `-x` flag, which stops test execution on the first failure—a powerful technique for focused debugging.

## The -x Flag (Stop on First Failure)

## Why Stop on First Failure?

When you have a test suite with multiple failures, pytest shows you all of them. This is useful for getting a complete picture, but when debugging, you often want to focus on one problem at a time.

The `-x` flag (or `--exitfirst`) tells pytest to stop immediately after the first test failure. This is invaluable for:

1. **Focused debugging**: Fix one problem before moving to the next
2. **Fast feedback loops**: Don't wait for the entire suite to run
3. **Cascading failures**: When one failure causes many others
4. **Development workflow**: Iterate quickly on a single failing test

Let's see this in action with our payment processor.

```python
# test_payment_suite.py
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_payment_with_valid_data():
    """Test successful payment processing."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    assert transaction_id == 'TXN_1'

def test_payment_fee_calculation():
    """Test that fees are calculated correctly."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    transaction = processor.get_transaction(transaction_id)
    # This will fail due to floating-point precision
    assert transaction['fee'] == 2.90

def test_payment_net_amount():
    """Test that net amount is calculated correctly."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    transaction = processor.get_transaction(transaction_id)
    # This will also fail due to floating-point precision
    assert transaction['net_amount'] == 97.10

def test_multiple_payments():
    """Test processing multiple payments."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    # Process three payments
    processor.process_payment(100.00, '1234567890123456', '123')
    processor.process_payment(200.00, '1234567890123456', '123')
    processor.process_payment(300.00, '1234567890123456', '123')
    
    # This will fail because we're checking exact equality
    total_fees = sum(t['fee'] for t in processor.transaction_log)
    assert total_fees == 17.40  # 2.9% of 600.00

def test_api_call_count():
    """Test that API is called correct number of times."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    processor.process_payment(100.00, '1234567890123456', '123')
    processor.process_payment(200.00, '1234567890123456', '123')
    
    assert api_client.call_count == 2
```

### Without -x: See All Failures

```bash
$ pytest test_payment_suite.py -v
```

**Output**:

```
============================= test session starts ==============================
collected 5 items

test_payment_suite.py::test_payment_with_valid_data PASSED              [ 20%]
test_payment_suite.py::test_payment_fee_calculation FAILED              [ 40%]
test_payment_suite.py::test_payment_net_amount FAILED                   [ 60%]
test_payment_suite.py::test_multiple_payments FAILED                    [ 80%]
test_payment_suite.py::test_api_call_count PASSED                       [100%]

=================================== FAILURES ===================================
__________________________ test_payment_fee_calculation ________________________
[full traceback for test_payment_fee_calculation]

___________________________ test_payment_net_amount ____________________________
[full traceback for test_payment_net_amount]

___________________________ test_multiple_payments _____________________________
[full traceback for test_multiple_payments]

========================= 2 passed, 3 failed in 0.05s ==========================
```

**What happened**:
- Pytest ran all 5 tests
- Showed 3 complete failure tracebacks
- You have to scroll through all failures to understand any single one

### With -x: Stop at First Failure

```bash
$ pytest test_payment_suite.py -v -x
```

**Output**:

```
============================= test session starts ==============================
collected 5 items

test_payment_suite.py::test_payment_with_valid_data PASSED              [ 20%]
test_payment_suite.py::test_payment_fee_calculation FAILED              [ 40%]

=================================== FAILURES ===================================
__________________________ test_payment_fee_calculation ________________________

    def test_payment_fee_calculation():
        """Test that fees are calculated correctly."""
        api_client = MockAPIClient()
        processor = PaymentProcessor(api_client)
        
        transaction_id = processor.process_payment(
            amount=100.00,
            card_number='1234567890123456',
            cvv='123'
        )
        
        transaction = processor.get_transaction(transaction_id)
        # This will fail due to floating-point precision
>       assert transaction['fee'] == 2.90
E       AssertionError: assert 2.8999999999999995 == 2.90

test_payment_suite.py:31: AssertionError
!!!!!!!!!!!!!!!!!!!!!!! stopping after 1 failures !!!!!!!!!!!!!!!!!!!!!!!!!!!!!
========================= 1 passed, 1 failed in 0.02s ==========================
```

**What happened**:
- Pytest ran tests until the first failure
- Stopped immediately after `test_payment_fee_calculation` failed
- Showed only one failure traceback
- Notice the message: "stopping after 1 failures"
- Tests 3, 4, and 5 were never executed

### Diagnostic Analysis: The First Failure

**The failure**: Floating-point precision issue in fee calculation

**Root cause**: `100.00 * 0.029 = 2.8999999999999995` (not exactly `2.90`)

**Why -x helped**: 
- We can focus on this one problem
- We don't get distracted by other failures
- We can fix this and re-run quickly

**The fix**: Use `pytest.approx()` for floating-point comparisons

```python
# test_payment_suite.py (fixed)
def test_payment_fee_calculation():
    """Test that fees are calculated correctly."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    transaction = processor.get_transaction(transaction_id)
    # Use pytest.approx() for floating-point comparison
    assert transaction['fee'] == pytest.approx(2.90)
```

Now run again with `-x`:

```bash
$ pytest test_payment_suite.py -v -x
```

**Output**:

```
============================= test session starts ==============================
collected 5 items

test_payment_suite.py::test_payment_with_valid_data PASSED              [ 20%]
test_payment_suite.py::test_payment_fee_calculation PASSED              [ 40%]
test_payment_suite.py::test_payment_net_amount FAILED                   [ 60%]

=================================== FAILURES ===================================
___________________________ test_payment_net_amount ____________________________

    def test_payment_net_amount():
        """Test that net amount is calculated correctly."""
        api_client = MockAPIClient()
        processor = PaymentProcessor(api_client)
        
        transaction_id = processor.process_payment(
            amount=100.00,
            card_number='1234567890123456',
            cvv='123'
        )
        
        transaction = processor.get_transaction(transaction_id)
        # This will also fail due to floating-point precision
>       assert transaction['net_amount'] == 97.10
E       AssertionError: assert 97.10000000000001 == 97.10

test_payment_suite.py:48: AssertionError
!!!!!!!!!!!!!!!!!!!!!!! stopping after 1 failures !!!!!!!!!!!!!!!!!!!!!!!!!!!!!
========================= 1 passed, 2 failed in 0.02s ==========================
```

**Progress**: 
- First test now passes
- Stopped at the next failure
- Same type of problem (floating-point precision)
- We can apply the same fix

## The Iterative Fix Workflow

The `-x` flag enables a powerful iterative workflow:

### Iteration 1: Identify First Failure

```bash
$ pytest test_payment_suite.py -v -x
# Output: test_payment_fee_calculation FAILED
```

### Iteration 2: Fix First Failure

```python
# Fix: Use pytest.approx() in test_payment_fee_calculation
assert transaction['fee'] == pytest.approx(2.90)
```

### Iteration 3: Re-run, Find Next Failure

```bash
$ pytest test_payment_suite.py -v -x
# Output: test_payment_net_amount FAILED
```

### Iteration 4: Fix Second Failure

```python
# Fix: Use pytest.approx() in test_payment_net_amount
assert transaction['net_amount'] == pytest.approx(97.10)
```

### Iteration 5: Re-run, Find Next Failure

```bash
$ pytest test_payment_suite.py -v -x
# Output: test_multiple_payments FAILED
```

### Iteration 6: Fix Third Failure

```python
# Fix: Use pytest.approx() in test_multiple_payments
assert total_fees == pytest.approx(17.40)
```

### Iteration 7: Verify All Pass

```bash
$ pytest test_payment_suite.py -v
# Output: 5 passed in 0.02s
```

## Combining -x with Other Flags

### -x with -vv: Stop on First Failure with Full Detail

```bash
$ pytest test_payment_suite.py -x -vv
```

**When to use**: You want to stop at the first failure AND see full object comparisons.

### -x with -l: Stop on First Failure with Local Variables

```bash
$ pytest test_payment_suite.py -x -vv -l
```

**When to use**: You need to see the state of all variables when the first failure occurred.

### -x with --pdb: Stop and Debug Immediately

```bash
$ pytest test_payment_suite.py -x --pdb
```

**When to use**: You want to drop into the debugger at the first failure (covered in detail in section 17.4).

### -x with -k: Stop on First Failure in Filtered Tests

```bash
$ pytest test_payment_suite.py -x -k "fee or net"
```

**Output**:

```
collected 5 items / 3 deselected / 2 selected

test_payment_suite.py::test_payment_fee_calculation PASSED              [ 50%]
test_payment_suite.py::test_payment_net_amount FAILED                   [100%]

[failure details]
!!!!!!!!!!!!!!!!!!!!!!! stopping after 1 failures !!!!!!!!!!!!!!!!!!!!!!!!!!!!!
```

**When to use**: You're debugging a specific subset of tests and want to stop at the first failure in that subset.

## Advanced: --maxfail=N

The `-x` flag is actually shorthand for `--maxfail=1`. You can specify a different number:

```bash
# Stop after 2 failures
$ pytest test_payment_suite.py --maxfail=2 -v
```

**Output**:

```
test_payment_suite.py::test_payment_with_valid_data PASSED              [ 20%]
test_payment_suite.py::test_payment_fee_calculation FAILED              [ 40%]
test_payment_suite.py::test_payment_net_amount FAILED                   [ 60%]

[failure details for both tests]
!!!!!!!!!!!!!!!!!!!!!!! stopping after 2 failures !!!!!!!!!!!!!!!!!!!!!!!!!!!!!
========================= 1 passed, 2 failed in 0.03s ==========================
```

**When to use**:
- You want to see a few failures but not all
- You're investigating related failures
- You want to batch-fix similar issues

## Real-World Scenario: Cascading Failures

Let's see a scenario where `-x` is especially valuable: cascading failures where one root cause creates many symptoms.

```python
# payment_processor.py (introduce a bug)
class PaymentProcessor:
    def __init__(self, api_client):
        self.api_client = api_client
        self.transaction_log = []
        self.fee_percentage = 0.029
        # Bug: Initialize with wrong state
        self._initialized = False  # Should be True
    
    def process_payment(self, amount, card_number, cvv):
        """Process a payment and return transaction ID."""
        # Bug: Check initialization state
        if not self._initialized:
            raise RuntimeError("Processor not initialized")
        
        # ... rest of the method
```

```python
# test_payment_suite.py (all tests will fail now)
def test_payment_with_valid_data():
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    # This will fail: RuntimeError: Processor not initialized
    transaction_id = processor.process_payment(...)

def test_payment_fee_calculation():
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    # This will fail: RuntimeError: Processor not initialized
    transaction_id = processor.process_payment(...)

# ... all other tests will fail the same way
```

Without `-x`:

```bash
$ pytest test_payment_suite.py -v
```

**Output**:

```
test_payment_suite.py::test_payment_with_valid_data FAILED              [ 20%]
test_payment_suite.py::test_payment_fee_calculation FAILED              [ 40%]
test_payment_suite.py::test_payment_net_amount FAILED                   [ 60%]
test_payment_suite.py::test_multiple_payments FAILED                    [ 80%]
test_payment_suite.py::test_api_call_count FAILED                       [100%]

[5 identical failure tracebacks]
========================= 5 failed in 0.05s ==========================
```

**Problem**: You see 5 failures, but they're all the same root cause. You waste time reading 5 identical tracebacks.

With `-x`:

```bash
$ pytest test_payment_suite.py -v -x
```

**Output**:

```
test_payment_suite.py::test_payment_with_valid_data FAILED              [ 20%]

=================================== FAILURES ===================================
_________________________ test_payment_with_valid_data _________________________

    def test_payment_with_valid_data():
        api_client = MockAPIClient()
        processor = PaymentProcessor(api_client)
>       transaction_id = processor.process_payment(
            amount=100.00,
            card_number='1234567890123456',
            cvv='123'
        )

payment_processor.py:15: RuntimeError: Processor not initialized
!!!!!!!!!!!!!!!!!!!!!!! stopping after 1 failures !!!!!!!!!!!!!!!!!!!!!!!!!!!!!
========================= 1 failed in 0.01s ==========================
```

**Benefit**: You immediately see the root cause. Fix the initialization bug once, and all tests will pass.

### The Fix

```python
# payment_processor.py (fixed)
class PaymentProcessor:
    def __init__(self, api_client):
        self.api_client = api_client
        self.transaction_log = []
        self.fee_percentage = 0.029
        self._initialized = True  # Fixed: Initialize correctly
```

Now all tests pass:

```bash
$ pytest test_payment_suite.py -v
# Output: 5 passed in 0.02s
```

## When NOT to Use -x

### Scenario 1: Comprehensive Test Reports

When you need to see all failures for a complete picture:

```bash
# CI pipeline: Want to see all failures
$ pytest -v --tb=short
```

### Scenario 2: Independent Failures

When failures are unrelated and you want to fix multiple issues in one session:

```bash
# See all failures, fix them all at once
$ pytest -v
```

### Scenario 3: Test Coverage Analysis

When you're measuring coverage and need all tests to run:

```bash
# Coverage needs all tests to run
$ pytest --cov=mypackage
```

## Decision Framework: When to Use -x

| Scenario | Use -x? | Why |
|----------|---------|-----|
| Debugging a specific failure | ✅ Yes | Focus on one problem |
| Iterative development | ✅ Yes | Fast feedback loop |
| Cascading failures | ✅ Yes | Find root cause quickly |
| CI/CD pipeline | ❌ No | Need complete report |
| Coverage measurement | ❌ No | Need all tests to run |
| Investigating multiple issues | ❌ No | Want to see all failures |
| Test suite validation | ❌ No | Need comprehensive results |

## Practical Patterns

### Pattern 1: The Debug Loop

```bash
# 1. Run with -x to find first failure
$ pytest -x -vv

# 2. Fix the failure

# 3. Re-run with -x to find next failure
$ pytest -x -vv

# 4. Repeat until all pass

# 5. Final verification without -x
$ pytest -v
```

### Pattern 2: The Focused Investigation

```bash
# Focus on a specific test file
$ pytest test_specific.py -x -vv -l

# Even more focused: specific test
$ pytest test_specific.py::test_name -x -vv -l --pdb
```

### Pattern 3: The Batch Fix

```bash
# See first N failures
$ pytest --maxfail=3 -v

# Fix all similar issues

# Verify all pass
$ pytest -v
```

## Key Takeaways: The -x Flag

### 1. -x Enables Focused Debugging
Stop at the first failure to concentrate on one problem at a time.

### 2. Perfect for Iterative Development
Fix one issue, re-run, fix the next. Fast feedback loops accelerate development.

### 3. Reveals Root Causes in Cascading Failures
When one bug causes many test failures, -x shows you the root cause immediately.

### 4. Combine with Other Flags for Maximum Effectiveness
`-x -vv -l` gives you focused, detailed debugging information.

### 5. --maxfail=N Provides Flexibility
Stop after N failures instead of just 1 when you want to see related issues.

### 6. Not for CI/CD
In continuous integration, you want comprehensive reports, not early exits.

### 7. The Debug Loop is Your Friend
Run with -x, fix, re-run with -x, fix, repeat. This workflow is incredibly efficient.

In the next section, we'll explore the `--pdb` flag, which takes debugging to the next level by dropping you into an interactive debugger at the point of failure.

## The --pdb Flag (Drop into Debugger)

## Interactive Debugging at the Point of Failure

The `--pdb` flag is one of pytest's most powerful debugging features. When a test fails, instead of just showing you the traceback, pytest drops you into Python's interactive debugger (pdb) at the exact point of failure. You can inspect variables, execute code, and understand exactly what went wrong.

This transforms debugging from "read the error and guess" to "explore the failure state interactively."

### What is pdb?

`pdb` is Python's built-in debugger. It provides an interactive shell where you can:
- Inspect variable values
- Execute Python code
- Step through code line by line
- Navigate the call stack
- Modify variables and continue execution

When pytest integrates with pdb via `--pdb`, it automatically drops you into the debugger when a test fails, with the execution context preserved.

Let's see this in action with our payment processor.

```python
# test_payment_debugging.py
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_payment_with_complex_calculation():
    """Test payment processing with multiple calculations."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    # Process a payment
    amount = 1234.56
    transaction_id = processor.process_payment(
        amount=amount,
        card_number='1234567890123456',
        cvv='123'
    )
    
    # Get the transaction
    transaction = processor.get_transaction(transaction_id)
    
    # Calculate expected values
    expected_fee = amount * 0.029
    expected_net = amount - expected_fee
    
    # These assertions will fail due to floating-point precision
    assert transaction['fee'] == expected_fee
    assert transaction['net_amount'] == expected_net
```

### Running with --pdb

```bash
$ pytest test_payment_debugging.py::test_payment_with_complex_calculation --pdb
```

**Output**:

```
============================= test session starts ==============================
collected 1 item

test_payment_debugging.py::test_payment_with_complex_calculation FAILED

=================================== FAILURES ===================================
_________________ test_payment_with_complex_calculation ________________________

    def test_payment_with_complex_calculation():
        """Test payment processing with multiple calculations."""
        api_client = MockAPIClient()
        processor = PaymentProcessor(api_client)
        
        # Process a payment
        amount = 1234.56
        transaction_id = processor.process_payment(
            amount=amount,
            card_number='1234567890123456',
            cvv='123'
        )
        
        # Get the transaction
        transaction = processor.get_transaction(transaction_id)
        
        # Calculate expected values
        expected_fee = amount * 0.029
        expected_net = amount - expected_fee
        
        # These assertions will fail due to floating-point precision
>       assert transaction['fee'] == expected_fee
E       AssertionError: assert 35.80224 == 35.80223999999999

test_payment_debugging.py:24: AssertionError
>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> entering PDB >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>

>>>>>>>>>>>>>> PDB post_mortem (IO-capturing turned off) >>>>>>>>>>>>>>>>>>>>>>
> /home/user/project/test_payment_debugging.py(24)test_payment_with_complex_calculation()
-> assert transaction['fee'] == expected_fee
(Pdb) 
```

**What happened**:
- The test failed at the assertion
- Pytest automatically entered pdb
- You're now at an interactive prompt: `(Pdb)`
- The execution is paused at the exact line that failed
- You can now explore the state

### Diagnostic Analysis: Exploring the Failure Interactively

Now we're in the debugger. Let's explore what went wrong.

**Step 1: Inspect the variables involved in the assertion**

```python
(Pdb) transaction['fee']
35.80224
(Pdb) expected_fee
35.80223999999999
(Pdb) transaction['fee'] == expected_fee
False
```

**What this tells us**: The values are almost identical but not exactly equal due to floating-point precision.

**Step 2: See the full transaction object**

```python
(Pdb) transaction
{'id': 'TXN_1', 'amount': 1234.56, 'fee': 35.80224, 'net_amount': 1198.7577600000001, 'status': 'completed'}
```

**What this tells us**: We can see all fields in the transaction, not just the one that failed.

**Step 3: Check the calculation**

```python
(Pdb) amount
1234.56
(Pdb) amount * 0.029
35.80223999999999
(Pdb) processor.fee_percentage
0.029
```

**What this tells us**: The calculation is correct, but floating-point arithmetic introduces tiny errors.

**Step 4: Test a potential fix**

```python
(Pdb) import pytest
(Pdb) transaction['fee'] == pytest.approx(expected_fee)
True
```

**What this tells us**: Using `pytest.approx()` would make the assertion pass.

**Step 5: Exit the debugger**

```python
(Pdb) quit
```

### Essential pdb Commands

When you're in the debugger, these commands are your tools:

#### Inspection Commands

**`p` or `print`**: Print a variable's value

```python
(Pdb) p transaction
{'id': 'TXN_1', 'amount': 1234.56, ...}

(Pdb) p transaction['fee']
35.80224
```

**`pp`**: Pretty-print (better formatting for complex objects)

```python
(Pdb) pp transaction
{'amount': 1234.56,
 'fee': 35.80224,
 'id': 'TXN_1',
 'net_amount': 1198.7577600000001,
 'status': 'completed'}
```

**`type()`**: Check the type of a variable

```python
(Pdb) type(transaction)
<class 'dict'>

(Pdb) type(transaction['fee'])
<class 'float'>
```

**`dir()`**: List all attributes and methods

```python
(Pdb) dir(processor)
['__class__', '__delattr__', ..., 'api_client', 'fee_percentage', 'get_transaction', 'process_payment', 'transaction_log']
```

**`locals()`**: Show all local variables

```python
(Pdb) locals()
{'api_client': <payment_processor.MockAPIClient object at 0x...>,
 'processor': <payment_processor.PaymentProcessor object at 0x...>,
 'amount': 1234.56,
 'transaction_id': 'TXN_1',
 'transaction': {...},
 'expected_fee': 35.80223999999999,
 'expected_net': 1198.7577600000001}
```

#### Navigation Commands

**`l` or `list`**: Show the current code context

```python
(Pdb) l
 19         # Calculate expected values
 20         expected_fee = amount * 0.029
 21         expected_net = amount - expected_fee
 22         
 23         # These assertions will fail due to floating-point precision
 24  ->     assert transaction['fee'] == expected_fee
 25         assert transaction['net_amount'] == expected_net
[EOF]
```

**`ll` or `longlist`**: Show the entire function

```python
(Pdb) ll
  3     def test_payment_with_complex_calculation():
  4         """Test payment processing with multiple calculations."""
  5         api_client = MockAPIClient()
  6         processor = PaymentProcessor(api_client)
  7         
  8         # Process a payment
  9         amount = 1234.56
 10         transaction_id = processor.process_payment(
 11             amount=amount,
 12             card_number='1234567890123456',
 13             cvv='123'
 14         )
 15         
 16         # Get the transaction
 17         transaction = processor.get_transaction(transaction_id)
 18         
 19         # Calculate expected values
 20         expected_fee = amount * 0.029
 21         expected_net = amount - expected_fee
 22         
 23         # These assertions will fail due to floating-point precision
 24  ->     assert transaction['fee'] == expected_fee
 25         assert transaction['net_amount'] == expected_net
```

**`w` or `where`**: Show the call stack

```python
(Pdb) w
  /usr/lib/python3.11/site-packages/_pytest/python.py(194)pytest_pyfunc_call()
-> result = testfunction(**testargs)
  /home/user/project/test_payment_debugging.py(24)test_payment_with_complex_calculation()
-> assert transaction['fee'] == expected_fee
```

**`u` or `up`**: Move up one level in the call stack

```python
(Pdb) u
> /usr/lib/python3.11/site-packages/_pytest/python.py(194)pytest_pyfunc_call()
-> result = testfunction(**testargs)
```

**`d` or `down`**: Move down one level in the call stack

```python
(Pdb) d
> /home/user/project/test_payment_debugging.py(24)test_payment_with_complex_calculation()
-> assert transaction['fee'] == expected_fee
```

#### Execution Commands

**`c` or `continue`**: Continue execution until the next breakpoint or failure

```python
(Pdb) c
# Continues execution, may hit another failure or exit
```

**`q` or `quit`**: Exit the debugger and stop test execution

```python
(Pdb) q
# Exits pdb and pytest
```

**`!` prefix**: Execute arbitrary Python code

```python
(Pdb) !import math
(Pdb) !math.isclose(transaction['fee'], expected_fee)
True

(Pdb) !new_amount = 500.00
(Pdb) !new_amount * 0.029
14.5
```

## Combining --pdb with Other Flags

### --pdb with -x: Debug First Failure Only

```bash
$ pytest test_payment_debugging.py -x --pdb
```

**When to use**: You want to debug the first failure and ignore subsequent ones.

### --pdb with -v: Verbose Output Before Debugging

```bash
$ pytest test_payment_debugging.py -v --pdb
```

**When to use**: You want to see test names and progress before dropping into the debugger.

### --pdb with -k: Debug Specific Tests

```bash
$ pytest -k "complex_calculation" --pdb
```

**When to use**: You want to debug only tests matching a pattern.

### --pdb with --lf: Debug Last Failed Test

```bash
$ pytest --lf --pdb
```

**When to use**: You want to re-run and debug the test that failed in the previous run.

## Advanced: --pdbcls for Better Debuggers

Pytest supports alternative debuggers that provide enhanced features:

### Using ipdb (IPython Debugger)

Install ipdb:

```bash
$ pip install ipdb
```

Use it with pytest:

```bash
$ pytest --pdb --pdbcls=IPython.terminal.debugger:TerminalPdb
```

**Benefits of ipdb**:
- Syntax highlighting
- Tab completion
- Better history
- IPython magic commands

### Using pdb++

Install pdb++:

```bash
$ pip install pdbpp
```

Use it automatically (it replaces pdb):

```bash
$ pytest --pdb
```

**Benefits of pdb++**:
- Syntax highlighting
- Tab completion
- Sticky mode (shows code context automatically)
- Better command shortcuts

## Real-World Debugging Scenario

Let's work through a complex debugging scenario that demonstrates the power of `--pdb`.

```python
# test_complex_debugging.py
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_refund_processing():
    """Test that refunds are processed correctly."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    # Process original payment
    original_amount = 500.00
    transaction_id = processor.process_payment(
        amount=original_amount,
        card_number='1234567890123456',
        cvv='123'
    )
    
    # Process refund (not implemented yet)
    refund_amount = 250.00
    refund_id = processor.process_refund(transaction_id, refund_amount)
    
    # Verify refund
    refund = processor.get_transaction(refund_id)
    assert refund['type'] == 'refund'
    assert refund['amount'] == refund_amount
    assert refund['original_transaction'] == transaction_id
```

This test will fail because `process_refund()` doesn't exist yet. Let's debug it:

```bash
$ pytest test_complex_debugging.py::test_refund_processing --pdb
```

**Output**:

```
=================================== FAILURES ===================================
__________________________ test_refund_processing ______________________________

    def test_refund_processing():
        """Test that refunds are processed correctly."""
        api_client = MockAPIClient()
        processor = PaymentProcessor(api_client)
        
        # Process original payment
        original_amount = 500.00
        transaction_id = processor.process_payment(
            amount=original_amount,
            card_number='1234567890123456',
            cvv='123'
        )
        
        # Process refund (not implemented yet)
        refund_amount = 250.00
>       refund_id = processor.process_refund(transaction_id, refund_amount)
E       AttributeError: 'PaymentProcessor' object has no attribute 'process_refund'

>>>>>>>>>>>>>>>>>>>>>>>>>>>>>> entering PDB >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>
(Pdb) 
```

### Interactive Debugging Session

**Step 1: Understand what we have**

```python
(Pdb) processor
<payment_processor.PaymentProcessor object at 0x7f8b3c4d5e90>

(Pdb) dir(processor)
['__class__', ..., 'api_client', 'fee_percentage', 'get_transaction', 'process_payment', 'transaction_log']

(Pdb) pp processor.transaction_log
[{'amount': 500.0,
  'fee': 14.499999999999998,
  'id': 'TXN_1',
  'net_amount': 485.50000000000006,
  'status': 'completed'}]
```

**What we learned**: 
- The original payment was processed successfully
- We have the transaction in the log
- The `process_refund` method doesn't exist

**Step 2: Check what the test expects**

```python
(Pdb) transaction_id
'TXN_1'

(Pdb) refund_amount
250.0

(Pdb) original_amount
500.0
```

**What we learned**: The test wants to refund $250 from a $500 transaction.

**Step 3: Verify the original transaction**

```python
(Pdb) original_transaction = processor.get_transaction(transaction_id)
(Pdb) pp original_transaction
{'amount': 500.0,
 'fee': 14.499999999999998,
 'id': 'TXN_1',
 'net_amount': 485.50000000000006,
 'status': 'completed'}
```

**What we learned**: The original transaction exists and has all the data we need.

**Step 4: Exit and implement the missing method**

```python
(Pdb) quit
```

Now implement `process_refund()`:

```python
# payment_processor.py (add this method)
class PaymentProcessor:
    # ... existing code ...
    
    def process_refund(self, original_transaction_id, refund_amount):
        """Process a refund for a previous transaction."""
        # Find the original transaction
        original = self.get_transaction(original_transaction_id)
        if original is None:
            raise ValueError(f"Transaction {original_transaction_id} not found")
        
        if refund_amount > original['amount']:
            raise ValueError("Refund amount exceeds original transaction")
        
        # Calculate refund fee (same percentage)
        refund_fee = refund_amount * self.fee_percentage
        refund_net = refund_amount - refund_fee
        
        # Create refund transaction
        refund_id = f"REFUND_{len(self.transaction_log) + 1}"
        self.transaction_log.append({
            'id': refund_id,
            'type': 'refund',
            'amount': refund_amount,
            'fee': refund_fee,
            'net_amount': refund_net,
            'original_transaction': original_transaction_id,
            'status': 'completed'
        })
        
        return refund_id
```

Run the test again:

```bash
$ pytest test_complex_debugging.py::test_refund_processing -v
```

**Output**:

```
test_complex_debugging.py::test_refund_processing PASSED                [100%]
```

Success! The interactive debugging session helped us:
1. Understand what data was available
2. Verify the test's expectations
3. Identify exactly what was missing
4. Implement the solution with confidence

## Practical Debugging Patterns

### Pattern 1: The Inspection Loop

```python
# In pdb:
(Pdb) p variable_name          # Check value
(Pdb) type(variable_name)      # Check type
(Pdb) dir(variable_name)       # Check available methods
(Pdb) pp variable_name         # Pretty print if complex
```

### Pattern 2: The Hypothesis Test

```python
# In pdb:
(Pdb) !import pytest
(Pdb) !pytest.approx(actual) == expected  # Test if approx would work
(Pdb) !len(actual) == len(expected)       # Test if length matches
(Pdb) !set(actual) == set(expected)       # Test if sets are equal
```

### Pattern 3: The State Exploration

```python
# In pdb:
(Pdb) locals()                 # See all local variables
(Pdb) pp self.__dict__         # See object state
(Pdb) w                        # See call stack
(Pdb) u                        # Go up to caller
(Pdb) pp locals()              # See caller's variables
```

### Pattern 4: The Quick Fix Verification

```python
# In pdb:
(Pdb) !fixed_value = actual.strip()  # Try a fix
(Pdb) !fixed_value == expected       # Test if it works
(Pdb) quit                           # Exit and implement the fix
```

## When to Use --pdb

### ✅ Use --pdb When:

1. **The failure is mysterious**: The traceback doesn't make the problem obvious
2. **You need to inspect state**: You want to see variable values at the failure point
3. **You're exploring**: You don't know what's wrong and need to investigate
4. **You want to test hypotheses**: You want to try different fixes interactively
5. **Complex data structures**: You need to navigate nested objects or large dictionaries

### ❌ Don't Use --pdb When:

1. **The failure is obvious**: The traceback clearly shows the problem
2. **Batch processing**: You're running many tests in CI/CD
3. **You know the fix**: You already understand what's wrong
4. **Automated environments**: CI servers can't interact with pdb
5. **Quick iterations**: Adding print statements might be faster

## Decision Framework: --pdb vs. Other Debugging Methods

| Debugging Need | Best Approach | Why |
|----------------|---------------|-----|
| Inspect variable at failure | `--pdb` | Interactive exploration |
| See all local variables | `--pdb` or `-l` | `-l` is faster if you don't need interaction |
| Test a hypothesis | `--pdb` | Can execute code interactively |
| Understand call stack | `--pdb` with `w`, `u`, `d` | Navigate the stack interactively |
| Quick value check | `-vv -l` | Faster than interactive debugging |
| Complex object inspection | `--pdb` with `pp` | Pretty-print complex structures |
| Automated debugging | `-vv -l --tb=long` | Can't use interactive debugger in CI |

## Key Takeaways: The --pdb Flag

### 1. --pdb Transforms Debugging from Passive to Active
Instead of reading error messages, you actively explore the failure state.

### 2. Essential pdb Commands: p, pp, l, w, q
Master these five commands and you can debug most issues.

### 3. Combine with -x for Focused Debugging
`-x --pdb` stops at the first failure and drops you into the debugger.

### 4. Use ! to Execute Arbitrary Code
Test hypotheses, try fixes, and explore possibilities interactively.

### 5. Navigate the Call Stack with u and d
Understand the execution context by moving up and down the stack.

### 6. ipdb and pdb++ Provide Better Experience
Consider upgrading to enhanced debuggers for syntax highlighting and tab completion.

### 7. Not for Automated Environments
`--pdb` requires human interaction, so it's not suitable for CI/CD.

In the next section, we'll explore how to use breakpoints in your tests for even more control over debugging.

## Using Breakpoints in Tests

## Proactive Debugging with Breakpoints

While `--pdb` drops you into the debugger when a test fails, sometimes you want to pause execution at a specific point before any failure occurs. This is where breakpoints come in.

A breakpoint is a deliberate pause point in your code where execution stops and you can inspect the state. Python 3.7+ provides the built-in `breakpoint()` function, which integrates seamlessly with pytest.

### Why Use Breakpoints?

1. **Inspect state before failure**: See what's happening before the assertion fails
2. **Understand execution flow**: Verify that code paths are executed as expected
3. **Debug complex logic**: Step through complicated calculations or transformations
4. **Validate assumptions**: Check that your mental model matches reality
5. **Explore APIs**: Understand how external libraries behave

Let's explore breakpoints with our payment processor.

```python
# test_breakpoint_demo.py
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_payment_with_breakpoint():
    """Test payment processing with a breakpoint for inspection."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    amount = 1000.00
    
    # Set a breakpoint before processing
    breakpoint()
    
    transaction_id = processor.process_payment(
        amount=amount,
        card_number='1234567890123456',
        cvv='123'
    )
    
    # Set another breakpoint after processing
    breakpoint()
    
    transaction = processor.get_transaction(transaction_id)
    
    # Verify the transaction
    assert transaction['amount'] == amount
    assert transaction['status'] == 'completed'
```

### Running Tests with Breakpoints

```bash
$ pytest test_breakpoint_demo.py::test_payment_with_breakpoint -s
```

**Note**: The `-s` flag is important—it disables output capturing so you can interact with the debugger.

**Output**:

```
============================= test session starts ==============================
collected 1 item

test_breakpoint_demo.py::test_payment_with_breakpoint 
> /home/user/project/test_breakpoint_demo.py(12)test_payment_with_breakpoint()
-> transaction_id = processor.process_payment(
(Pdb) 
```

**What happened**:
- Execution stopped at the first `breakpoint()`
- You're in pdb at line 12, before `process_payment()` is called
- You can now inspect the state before the payment is processed

### Diagnostic Analysis: Exploring State at Breakpoints

**At the first breakpoint (before processing)**:

```python
(Pdb) amount
1000.0

(Pdb) processor.transaction_log
[]

(Pdb) api_client.call_count
0

(Pdb) ll
  3     def test_payment_with_breakpoint():
  4         """Test payment processing with a breakpoint for inspection."""
  5         api_client = MockAPIClient()
  6         processor = PaymentProcessor(api_client)
  7         
  8         amount = 1000.00
  9         
 10         # Set a breakpoint before processing
 11         breakpoint()
 12         
 13  ->     transaction_id = processor.process_payment(
 14             amount=amount,
 15             card_number='1234567890123456',
 16             cvv='123'
 17         )
 18         
 19         # Set another breakpoint after processing
 20         breakpoint()
```

**What we learned**:
- The amount is set correctly
- No transactions have been processed yet
- The API hasn't been called yet
- We're about to call `process_payment()`

**Continue to the next breakpoint**:

```python
(Pdb) c
> /home/user/project/test_breakpoint_demo.py(22)test_payment_with_breakpoint()
-> transaction = processor.get_transaction(transaction_id)
(Pdb)
```

**At the second breakpoint (after processing)**:

```python
(Pdb) transaction_id
'TXN_1'

(Pdb) pp processor.transaction_log
[{'amount': 1000.0,
  'fee': 28.999999999999996,
  'id': 'TXN_1',
  'net_amount': 971.0000000000001,
  'status': 'completed'}]

(Pdb) api_client.call_count
1

(Pdb) c
# Test continues and completes
```

**What we learned**:
- The payment was processed successfully
- Transaction ID is 'TXN_1'
- The transaction is in the log
- The API was called once
- We can now verify the transaction details

## Strategic Breakpoint Placement

### Pattern 1: Before and After Critical Operations

Place breakpoints around important operations to verify state changes:
    </markdown>

    
def test_multiple_payments_with_breakpoints():
    """Test multiple payments with strategic breakpoints."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    # Breakpoint: Initial state
    breakpoint()
    
    # Process first payment
    txn1 = processor.process_payment(100.00, '1234567890123456', '123')
    
    # Breakpoint: After first payment
    breakpoint()
    
    # Process second payment
    txn2 = processor.process_payment(200.00, '1234567890123456', '123')
    
    # Breakpoint: After second payment
    breakpoint()
    
    # Verify both transactions
    assert len(processor.transaction_log) == 2
    assert txn1 != txn2
```

**When to use**: When you need to verify that each step produces the expected state change.

### Pattern 2: Inside Loops

Place breakpoints inside loops to inspect iteration state:

```python
def test_batch_processing_with_breakpoint():
    """Test batch payment processing with loop inspection."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    amounts = [100.00, 200.00, 300.00, 400.00, 500.00]
    transaction_ids = []
    
    for i, amount in enumerate(amounts):
        # Breakpoint: Inspect each iteration
        if i == 2:  # Stop at third iteration
            breakpoint()
        
        txn_id = processor.process_payment(
            amount=amount,
            card_number='1234567890123456',
            cvv='123'
        )
        transaction_ids.append(txn_id)
    
    assert len(transaction_ids) == len(amounts)
```

**When to use**: When you need to understand what's happening in a specific iteration.

### Pattern 3: Conditional Breakpoints

Use conditional logic to break only when certain conditions are met:

```python
def test_conditional_breakpoint():
    """Test with conditional breakpoint."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    amounts = [50.00, 100.00, 150.00, 200.00, 250.00]
    
    for amount in amounts:
        transaction_id = processor.process_payment(
            amount=amount,
            card_number='1234567890123456',
            cvv='123'
        )
        
        transaction = processor.get_transaction(transaction_id)
        
        # Break only if fee is above a threshold
        if transaction['fee'] > 5.00:
            breakpoint()
        
        assert transaction['status'] == 'completed'
```

**When to use**: When you want to inspect state only in specific scenarios.

### Pattern 4: Before Assertions

Place breakpoints right before assertions to verify expected values:

```python
def test_assertion_inspection():
    """Test with breakpoint before assertion."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    transaction_id = processor.process_payment(
        amount=1234.56,
        card_number='1234567890123456',
        cvv='123'
    )
    
    transaction = processor.get_transaction(transaction_id)
    
    # Calculate expected values
    expected_fee = 1234.56 * 0.029
    expected_net = 1234.56 - expected_fee
    
    # Breakpoint: Verify calculations before asserting
    breakpoint()
    
    assert transaction['fee'] == pytest.approx(expected_fee)
    assert transaction['net_amount'] == pytest.approx(expected_net)
```

**When to use**: When you want to verify your expected values are correct before the assertion fails.

## Advanced Breakpoint Techniques

### Using breakpoint() with Arguments

The `breakpoint()` function accepts arguments that are passed to the debugger:

```python
def test_breakpoint_with_context():
    """Test with contextual breakpoint."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    amount = 500.00
    
    # You can't pass arguments to breakpoint() directly,
    # but you can set variables to provide context
    debug_context = {
        'step': 'before_payment',
        'amount': amount,
        'expected_fee': amount * 0.029
    }
    
    breakpoint()  # Inspect debug_context in pdb
    
    transaction_id = processor.process_payment(
        amount=amount,
        card_number='1234567890123456',
        cvv='123'
    )
    
    assert transaction_id is not None
```

### Disabling Breakpoints with Environment Variables

You can disable all breakpoints by setting the `PYTHONBREAKPOINT` environment variable:

```bash
# Disable all breakpoints
$ PYTHONBREAKPOINT=0 pytest test_breakpoint_demo.py

# Use a different debugger
$ PYTHONBREAKPOINT=ipdb.set_trace pytest test_breakpoint_demo.py
```

**When to use**: 
- In CI/CD pipelines where breakpoints should be ignored
- When you want to run tests without stopping at breakpoints
- When switching between different debuggers

### Temporary Breakpoints for Quick Debugging

Add breakpoints temporarily during development and remove them before committing:

```python
def test_temporary_debugging():
    """Test with temporary breakpoint for debugging."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    # TODO: Remove this breakpoint before committing
    breakpoint()
    
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    assert transaction_id is not None
```

**Best practice**: Use a linter or pre-commit hook to catch breakpoints before they reach version control.

## Breakpoints vs. --pdb: When to Use Each

### Use breakpoint() When:

1. **You know where to look**: You have a specific location you want to inspect
2. **Proactive debugging**: You want to understand code flow before any failure
3. **Iterative development**: You're building new functionality and want to verify each step
4. **Complex logic**: You need to step through complicated calculations
5. **Reusable inspection points**: You want to keep the breakpoint for multiple test runs

### Use --pdb When:

1. **Reactive debugging**: A test failed and you want to inspect the failure point
2. **Unknown failure location**: You don't know where the problem is
3. **One-time investigation**: You don't need a permanent breakpoint
4. **Multiple tests**: You want to debug any test that fails
5. **Quick inspection**: You want to drop into the debugger without modifying code

### Combining Both Approaches

You can use both `breakpoint()` and `--pdb` together:

```python
def test_combined_debugging():
    """Test using both breakpoint() and --pdb."""
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    # Breakpoint: Inspect initial state
    breakpoint()
    
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    transaction = processor.get_transaction(transaction_id)
    
    # This assertion might fail, triggering --pdb
    assert transaction['fee'] == 2.90  # Will fail without pytest.approx()
```

```bash
$ pytest test_combined_debugging.py -s --pdb
```

**What happens**:
1. Execution stops at `breakpoint()`
2. You inspect the initial state
3. You continue with `c`
4. If the assertion fails, `--pdb` drops you into the debugger again
5. You can inspect the failure state

## Real-World Debugging Scenario

Let's work through a complex scenario that demonstrates strategic breakpoint placement.

```python
# test_complex_breakpoint_scenario.py
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_payment_retry_logic():
    """Test payment retry logic with breakpoints."""
    # Simulate an API that fails twice then succeeds
    api_client = MockAPIClient(should_fail=True)
    processor = PaymentProcessor(api_client)
    
    amount = 100.00
    max_retries = 3
    retry_count = 0
    transaction_id = None
    
    # Breakpoint: Before retry loop
    breakpoint()
    
    while retry_count < max_retries:
        try:
            # Breakpoint: At start of each retry
            if retry_count > 0:
                breakpoint()
            
            transaction_id = processor.process_payment(
                amount=amount,
                card_number='1234567890123456',
                cvv='123'
            )
            
            # Success - break out of loop
            break
            
        except ConnectionError as e:
            retry_count += 1
            
            # Breakpoint: After each failure
            breakpoint()
            
            if retry_count >= max_retries:
                raise
            
            # Simulate fixing the API after 2 failures
            if retry_count == 2:
                api_client.should_fail = False
    
    # Breakpoint: After successful payment
    breakpoint()
    
    assert transaction_id is not None
    assert retry_count == 2  # Should succeed on third try
```

### Debugging Session Walkthrough

Run the test:

```bash
$ pytest test_complex_breakpoint_scenario.py::test_payment_retry_logic -s
```

**At first breakpoint (before retry loop)**:

```python
(Pdb) retry_count
0
(Pdb) api_client.should_fail
True
(Pdb) transaction_id
None
(Pdb) c
```

**At second breakpoint (after first failure)**:

```python
(Pdb) retry_count
1
(Pdb) api_client.should_fail
True
(Pdb) transaction_id
None
(Pdb) c
```

**At third breakpoint (start of second retry)**:

```python
(Pdb) retry_count
1
(Pdb) api_client.should_fail
True
(Pdb) c
```

**At fourth breakpoint (after second failure)**:

```python
(Pdb) retry_count
2
(Pdb) api_client.should_fail
False  # API is now fixed
(Pdb) c
```

**At fifth breakpoint (start of third retry)**:

```python
(Pdb) retry_count
2
(Pdb) api_client.should_fail
False
(Pdb) c
# Payment succeeds, loop breaks
```

**At sixth breakpoint (after successful payment)**:

```python
(Pdb) transaction_id
'TXN_1'
(Pdb) retry_count
2
(Pdb) api_client.call_count
3
(Pdb) c
# Test completes successfully
```

**What we learned**:
- The retry logic works correctly
- The API failed twice as expected
- The third attempt succeeded
- We verified the state at each critical point

## Practical Breakpoint Patterns

### Pattern 1: The State Verification Pattern

```python
def test_state_verification():
    # Setup
    obj = create_object()
    
    breakpoint()  # Verify initial state
    
    obj.modify()
    
    breakpoint()  # Verify state after modification
    
    assert obj.is_valid()
```

### Pattern 2: The Calculation Inspection Pattern

```python
def test_calculation_inspection():
    input_data = get_input()
    
    breakpoint()  # Inspect input
    
    intermediate = process_step_1(input_data)
    
    breakpoint()  # Inspect intermediate result
    
    final = process_step_2(intermediate)
    
    breakpoint()  # Inspect final result
    
    assert final == expected
```

### Pattern 3: The Conditional Investigation Pattern

```python
def test_conditional_investigation():
    for item in items:
        result = process(item)
        
        if result.is_suspicious():
            breakpoint()  # Investigate suspicious cases
        
        assert result.is_valid()
```

### Pattern 4: The Comparison Pattern

```python
def test_comparison():
    actual = compute_actual()
    expected = compute_expected()
    
    breakpoint()  # Compare actual vs expected before assertion
    
    assert actual == expected
```

## Best Practices for Using Breakpoints

### 1. Remove Breakpoints Before Committing

Use a pre-commit hook or linter to catch breakpoints:

```bash
# .pre-commit-config.yaml
repos:
  - repo: local
    hooks:
      - id: check-breakpoints
        name: Check for breakpoints
        entry: grep -r "breakpoint()" tests/
        language: system
        pass_filenames: false
```

### 2. Use Descriptive Variable Names Before Breakpoints

Make debugging easier by setting up context:

```python
def test_with_context():
    # Set up debugging context
    debug_info = {
        'step': 'before_critical_operation',
        'input': input_data,
        'expected': expected_result
    }
    
    breakpoint()  # Inspect debug_info
    
    result = critical_operation(input_data)
    assert result == expected_result
```

### 3. Combine with Logging

Use logging alongside breakpoints for permanent debugging information:

```python
import logging

def test_with_logging_and_breakpoint():
    logger = logging.getLogger(__name__)
    
    logger.info("Starting payment processing")
    
    transaction_id = processor.process_payment(...)
    
    logger.info(f"Transaction ID: {transaction_id}")
    
    breakpoint()  # Inspect if needed
    
    assert transaction_id is not None
```

### 4. Use Environment Variables for Conditional Breakpoints

Make breakpoints conditional on environment variables:

```python
import os

def test_conditional_breakpoint():
    if os.getenv('DEBUG'):
        breakpoint()
    
    result = process_data()
    assert result.is_valid()
```

```bash
# Enable debugging
$ DEBUG=1 pytest test_file.py -s

# Normal run (no breakpoints)
$ pytest test_file.py
```

## Key Takeaways: Using Breakpoints

### 1. breakpoint() Enables Proactive Debugging
Stop execution at specific points to inspect state before failures occur.

### 2. Strategic Placement is Key
Place breakpoints before/after critical operations, inside loops, or before assertions.

### 3. Combine with --pdb for Comprehensive Debugging
Use `breakpoint()` for planned inspection and `--pdb` for reactive debugging.

### 4. Use -s Flag to Enable Interaction
Always run with `-s` when using breakpoints to disable output capturing.

### 5. Conditional Breakpoints Reduce Noise
Use `if` statements to break only when specific conditions are met.

### 6. Remove Before Committing
Use pre-commit hooks or linters to prevent breakpoints from reaching production.

### 7. PYTHONBREAKPOINT=0 Disables All Breakpoints
Useful for CI/CD or when you want to run tests without stopping.

In the next section, we'll explore how to use logging and debugging information to create permanent debugging infrastructure in your tests.

## Logging and Debugging Information

## Building Permanent Debugging Infrastructure

While `--pdb` and `breakpoint()` are powerful for interactive debugging, they're temporary solutions. Logging provides permanent debugging infrastructure that helps you understand test behavior without stopping execution.

Logging is especially valuable for:
1. **Understanding test flow**: See what happens during test execution
2. **Debugging intermittent failures**: Capture information about flaky tests
3. **Performance analysis**: Track timing and resource usage
4. **Production debugging**: Understand behavior in CI/CD environments
5. **Historical analysis**: Review logs from past test runs

Let's build a comprehensive logging strategy for our payment processor.

```python
# payment_processor.py (with logging)
import logging

logger = logging.getLogger(__name__)

class PaymentProcessor:
    def __init__(self, api_client):
        self.api_client = api_client
        self.transaction_log = []
        self.fee_percentage = 0.029
        logger.info("PaymentProcessor initialized with fee_percentage=%.3f", 
                   self.fee_percentage)
    
    def process_payment(self, amount, card_number, cvv):
        """Process a payment and return transaction ID."""
        logger.debug("Processing payment: amount=%.2f, card=****%s, cvv=***",
                    amount, card_number[-4:])
        
        if amount <= 0:
            logger.error("Invalid amount: %.2f (must be positive)", amount)
            raise ValueError("Amount must be positive")
        
        if len(card_number) != 16:
            logger.error("Invalid card number length: %d (expected 16)", 
                        len(card_number))
            raise ValueError("Invalid card number")
        
        if len(str(cvv)) != 3:
            logger.error("Invalid CVV length: %d (expected 3)", len(str(cvv)))
            raise ValueError("Invalid CVV")
        
        # Calculate fee and net amount
        fee = amount * self.fee_percentage
        net_amount = amount - fee
        
        logger.debug("Calculated fee=%.2f, net_amount=%.2f", fee, net_amount)
        
        try:
            response = self.api_client.charge(
                amount=amount,
                card=card_number,
                cvv=cvv
            )
            logger.info("API charge successful: transaction_id=%s", 
                       response['transaction_id'])
        except Exception as e:
            logger.error("API charge failed: %s", str(e), exc_info=True)
            raise
        
        transaction_id = response['transaction_id']
        self.transaction_log.append({
            'id': transaction_id,
            'amount': amount,
            'fee': fee,
            'net_amount': net_amount,
            'status': 'completed'
        })
        
        logger.info("Payment processed successfully: transaction_id=%s, amount=%.2f",
                   transaction_id, amount)
        
        return transaction_id
    
    def get_transaction(self, transaction_id):
        """Retrieve a transaction by ID."""
        logger.debug("Looking up transaction: %s", transaction_id)
        
        for transaction in self.transaction_log:
            if transaction['id'] == transaction_id:
                logger.debug("Transaction found: %s", transaction_id)
                return transaction
        
        logger.warning("Transaction not found: %s", transaction_id)
        return None

class MockAPIClient:
    def __init__(self, should_fail=False):
        self.should_fail = should_fail
        self.call_count = 0
        logger.info("MockAPIClient initialized: should_fail=%s", should_fail)
    
    def charge(self, amount, card, cvv):
        self.call_count += 1
        logger.debug("API charge called (attempt %d): amount=%.2f", 
                    self.call_count, amount)
        
        if self.should_fail:
            logger.error("API charge failed (simulated failure)")
            raise ConnectionError("API unavailable")
        
        transaction_id = f'TXN_{self.call_count}'
        logger.info("API charge succeeded: transaction_id=%s", transaction_id)
        
        return {
            'transaction_id': transaction_id,
            'status': 'success'
        }
```

## Configuring Logging in Tests

### Basic Logging Configuration

```python
# test_logging_demo.py
import logging
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

# Configure logging for tests
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

def test_payment_with_logging():
    """Test payment processing with logging enabled."""
    logger = logging.getLogger(__name__)
    logger.info("=== Starting test_payment_with_logging ===")
    
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    logger.info("Processing payment for $100.00")
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    logger.info("Payment processed: transaction_id=%s", transaction_id)
    
    transaction = processor.get_transaction(transaction_id)
    logger.debug("Retrieved transaction: %s", transaction)
    
    assert transaction['amount'] == 100.00
    assert transaction['status'] == 'completed'
    
    logger.info("=== Test completed successfully ===")
```

Run the test with logging output:

```bash
$ pytest test_logging_demo.py::test_payment_with_logging -s
```

**Output**:

```
============================= test session starts ==============================
collected 1 item

test_logging_demo.py::test_payment_with_logging 
2024-01-15 10:30:45,123 - test_logging_demo - INFO - === Starting test_payment_with_logging ===
2024-01-15 10:30:45,124 - payment_processor - INFO - MockAPIClient initialized: should_fail=False
2024-01-15 10:30:45,124 - payment_processor - INFO - PaymentProcessor initialized with fee_percentage=0.029
2024-01-15 10:30:45,124 - test_logging_demo - INFO - Processing payment for $100.00
2024-01-15 10:30:45,125 - payment_processor - DEBUG - Processing payment: amount=100.00, card=****3456, cvv=***
2024-01-15 10:30:45,125 - payment_processor - DEBUG - Calculated fee=2.90, net_amount=97.10
2024-01-15 10:30:45,125 - payment_processor - DEBUG - API charge called (attempt 1): amount=100.00
2024-01-15 10:30:45,125 - payment_processor - INFO - API charge succeeded: transaction_id=TXN_1
2024-01-15 10:30:45,125 - payment_processor - INFO - API charge successful: transaction_id=TXN_1
2024-01-15 10:30:45,126 - payment_processor - INFO - Payment processed successfully: transaction_id=TXN_1, amount=100.00
2024-01-15 10:30:45,126 - test_logging_demo - INFO - Payment processed: transaction_id=TXN_1
2024-01-15 10:30:45,126 - payment_processor - DEBUG - Looking up transaction: TXN_1
2024-01-15 10:30:45,126 - payment_processor - DEBUG - Transaction found: TXN_1
2024-01-15 10:30:45,126 - test_logging_demo - DEBUG - Retrieved transaction: {'id': 'TXN_1', 'amount': 100.0, 'fee': 2.8999999999999995, 'net_amount': 97.10000000000001, 'status': 'completed'}
2024-01-15 10:30:45,126 - test_logging_demo - INFO - === Test completed successfully ===
PASSED

============================== 1 passed in 0.01s ===============================
```

**What we see**:
- Complete execution flow with timestamps
- All log levels (INFO, DEBUG)
- Detailed information about each operation
- Clear test boundaries

### Diagnostic Analysis: Reading Log Output

Let's trace through the log output to understand the execution flow:

**1. Test initialization**:
```
INFO - === Starting test_payment_with_logging ===
INFO - MockAPIClient initialized: should_fail=False
INFO - PaymentProcessor initialized with fee_percentage=0.029
```

**What this tells us**: The test started, and both objects were initialized correctly.

**2. Payment processing begins**:
```
INFO - Processing payment for $100.00
DEBUG - Processing payment: amount=100.00, card=****3456, cvv=***
DEBUG - Calculated fee=2.90, net_amount=97.10
```

**What this tells us**: The payment amount is correct, and fee calculation happened.

**3. API interaction**:
```
DEBUG - API charge called (attempt 1): amount=100.00
INFO - API charge succeeded: transaction_id=TXN_1
INFO - API charge successful: transaction_id=TXN_1
```

**What this tells us**: The API was called once and succeeded immediately.

**4. Transaction storage**:
```
INFO - Payment processed successfully: transaction_id=TXN_1, amount=100.00
```

**What this tells us**: The transaction was stored in the log.

**5. Transaction retrieval**:
```
DEBUG - Looking up transaction: TXN_1
DEBUG - Transaction found: TXN_1
DEBUG - Retrieved transaction: {...}
```

**What this tells us**: The transaction was successfully retrieved with all its data.

## Advanced Logging Configurations

### Using pytest's caplog Fixture

Pytest provides the `caplog` fixture for capturing and asserting on log messages:

```python
# test_caplog_demo.py
import logging
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_payment_logging_with_caplog(caplog):
    """Test that payment processing logs expected messages."""
    caplog.set_level(logging.INFO)
    
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    # Assert that specific log messages were generated
    assert "PaymentProcessor initialized" in caplog.text
    assert "Payment processed successfully" in caplog.text
    assert f"transaction_id={transaction_id}" in caplog.text
    
    # Check log levels
    assert any(record.levelname == "INFO" for record in caplog.records)
    
    # Check specific log records
    payment_logs = [r for r in caplog.records 
                   if "Payment processed successfully" in r.message]
    assert len(payment_logs) == 1
    assert payment_logs[0].levelname == "INFO"
```

**Benefits of caplog**:
- Capture logs without printing them
- Assert on log content
- Verify log levels
- Test logging behavior

### Logging Configuration with pytest.ini

```ini
# pytest.ini
[pytest]
log_cli = true
log_cli_level = INFO
log_cli_format = %(asctime)s [%(levelname)8s] %(name)s - %(message)s
log_cli_date_format = %Y-%m-%d %H:%M:%S

log_file = tests.log
log_file_level = DEBUG
log_file_format = %(asctime)s [%(levelname)8s] %(name)s - %(message)s (%(filename)s:%(lineno)d)
log_file_date_format = %Y-%m-%d %H:%M:%S
```

**What this does**:
- `log_cli = true`: Enable live logging to console
- `log_cli_level = INFO`: Show INFO and above in console
- `log_file = tests.log`: Write all logs to a file
- `log_file_level = DEBUG`: Include DEBUG logs in file

Now run tests:

```bash
$ pytest test_logging_demo.py
```

**Console output** (INFO and above):
```
2024-01-15 10:30:45 [    INFO] test_logging_demo - === Starting test_payment_with_logging ===
2024-01-15 10:30:45 [    INFO] payment_processor - MockAPIClient initialized: should_fail=False
2024-01-15 10:30:45 [    INFO] payment_processor - PaymentProcessor initialized with fee_percentage=0.029
...
```

**File output** (tests.log, includes DEBUG):
```
2024-01-15 10:30:45 [    INFO] test_logging_demo - === Starting test_payment_with_logging === (test_logging_demo.py:12)
2024-01-15 10:30:45 [    INFO] payment_processor - MockAPIClient initialized: should_fail=False (payment_processor.py:95)
2024-01-15 10:30:45 [    INFO] payment_processor - PaymentProcessor initialized with fee_percentage=0.029 (payment_processor.py:13)
2024-01-15 10:30:45 [   DEBUG] payment_processor - Processing payment: amount=100.00, card=****3456, cvv=*** (payment_processor.py:17)
...
```

### Structured Logging with JSON

For machine-readable logs, use JSON formatting:

```python
# test_structured_logging.py
import json
import logging
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

class JSONFormatter(logging.Formatter):
    """Custom formatter that outputs JSON."""
    
    def format(self, record):
        log_data = {
            'timestamp': self.formatTime(record),
            'level': record.levelname,
            'logger': record.name,
            'message': record.getMessage(),
            'module': record.module,
            'function': record.funcName,
            'line': record.lineno
        }
        
        # Add extra fields if present
        if hasattr(record, 'transaction_id'):
            log_data['transaction_id'] = record.transaction_id
        if hasattr(record, 'amount'):
            log_data['amount'] = record.amount
        
        return json.dumps(log_data)

def test_payment_with_structured_logging(caplog):
    """Test with structured JSON logging."""
    # Configure JSON formatter
    handler = logging.StreamHandler()
    handler.setFormatter(JSONFormatter())
    
    logger = logging.getLogger('payment_processor')
    logger.addHandler(handler)
    logger.setLevel(logging.DEBUG)
    
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    assert transaction_id is not None
```

**Output** (JSON format):
```json
{"timestamp": "2024-01-15 10:30:45,123", "level": "INFO", "logger": "payment_processor", "message": "MockAPIClient initialized: should_fail=False", "module": "payment_processor", "function": "__init__", "line": 95}
{"timestamp": "2024-01-15 10:30:45,124", "level": "INFO", "logger": "payment_processor", "message": "PaymentProcessor initialized with fee_percentage=0.029", "module": "payment_processor", "function": "__init__", "line": 13}
...
```

**Benefits**:
- Machine-readable format
- Easy to parse and analyze
- Can be ingested by log aggregation systems
- Structured data for queries

## Logging Strategies for Different Scenarios

### Strategy 1: Debug Intermittent Failures

For flaky tests, add detailed logging to capture state:

```python
# test_intermittent_failure.py
import logging
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_payment_with_retry_logging(caplog):
    """Test with detailed logging for retry logic."""
    caplog.set_level(logging.DEBUG)
    logger = logging.getLogger(__name__)
    
    api_client = MockAPIClient(should_fail=True)
    processor = PaymentProcessor(api_client)
    
    max_retries = 3
    retry_count = 0
    
    logger.info("Starting payment with retry logic: max_retries=%d", max_retries)
    
    while retry_count < max_retries:
        try:
            logger.info("Attempt %d of %d", retry_count + 1, max_retries)
            
            transaction_id = processor.process_payment(
                amount=100.00,
                card_number='1234567890123456',
                cvv='123'
            )
            
            logger.info("Payment succeeded on attempt %d", retry_count + 1)
            break
            
        except ConnectionError as e:
            retry_count += 1
            logger.warning("Payment failed on attempt %d: %s", retry_count, str(e))
            
            if retry_count >= max_retries:
                logger.error("All retry attempts exhausted")
                raise
            
            # Fix API after 2 failures
            if retry_count == 2:
                logger.info("Fixing API for next attempt")
                api_client.should_fail = False
    
    # Verify logging captured the retry behavior
    assert "Attempt 1 of 3" in caplog.text
    assert "Payment failed on attempt 1" in caplog.text
    assert "Attempt 2 of 3" in caplog.text
    assert "Payment failed on attempt 2" in caplog.text
    assert "Fixing API for next attempt" in caplog.text
    assert "Attempt 3 of 3" in caplog.text
    assert "Payment succeeded on attempt 3" in caplog.text
```

### Strategy 2: Performance Logging

Track timing information to identify slow operations:

```python
# test_performance_logging.py
import logging
import time
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_payment_performance_logging(caplog):
    """Test with performance timing logs."""
    caplog.set_level(logging.INFO)
    logger = logging.getLogger(__name__)
    
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    # Time the entire operation
    start_time = time.time()
    logger.info("Starting payment processing")
    
    # Time payment processing
    payment_start = time.time()
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    payment_duration = time.time() - payment_start
    logger.info("Payment processing took %.3f seconds", payment_duration)
    
    # Time transaction retrieval
    retrieval_start = time.time()
    transaction = processor.get_transaction(transaction_id)
    retrieval_duration = time.time() - retrieval_start
    logger.info("Transaction retrieval took %.3f seconds", retrieval_duration)
    
    total_duration = time.time() - start_time
    logger.info("Total operation took %.3f seconds", total_duration)
    
    assert transaction is not None
    
    # Assert performance expectations
    assert payment_duration < 1.0, "Payment processing too slow"
    assert retrieval_duration < 0.1, "Transaction retrieval too slow"
```

### Strategy 3: State Transition Logging

Log state changes to understand object lifecycle:

```python
# test_state_logging.py
import logging
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_payment_state_transitions(caplog):
    """Test with state transition logging."""
    caplog.set_level(logging.DEBUG)
    logger = logging.getLogger(__name__)
    
    logger.info("State: INITIAL - Creating objects")
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    logger.info("State: READY - Objects created, transaction_log=%d", 
               len(processor.transaction_log))
    
    logger.info("State: PROCESSING - Starting payment")
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    logger.info("State: PROCESSED - Payment complete, transaction_log=%d",
               len(processor.transaction_log))
    
    logger.info("State: RETRIEVING - Looking up transaction")
    transaction = processor.get_transaction(transaction_id)
    
    logger.info("State: COMPLETE - Transaction retrieved, status=%s",
               transaction['status'])
    
    # Verify state transitions were logged
    assert "State: INITIAL" in caplog.text
    assert "State: READY" in caplog.text
    assert "State: PROCESSING" in caplog.text
    assert "State: PROCESSED" in caplog.text
    assert "State: RETRIEVING" in caplog.text
    assert "State: COMPLETE" in caplog.text
```

### Strategy 4: Error Context Logging

Provide rich context when errors occur:

```python
# test_error_context_logging.py
import logging
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_payment_error_with_context(caplog):
    """Test error logging with rich context."""
    caplog.set_level(logging.DEBUG)
    logger = logging.getLogger(__name__)
    
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    # Attempt invalid payment
    invalid_amount = -50.00
    
    logger.info("Attempting payment with invalid data")
    logger.debug("Payment details: amount=%.2f, card=****3456, cvv=***",
                invalid_amount)
    
    try:
        transaction_id = processor.process_payment(
            amount=invalid_amount,
            card_number='1234567890123456',
            cvv='123'
        )
    except ValueError as e:
        logger.error("Payment validation failed: %s", str(e))
        logger.debug("Error context: amount=%.2f, processor_state=%s",
                    invalid_amount, 
                    {'transaction_count': len(processor.transaction_log)})
        
        # Verify error was logged with context
        assert "Payment validation failed" in caplog.text
        assert "Invalid amount" in caplog.text
        assert "Error context" in caplog.text
        
        return  # Expected error
    
    pytest.fail("Expected ValueError was not raised")
```

## Combining Logging with Other Debugging Tools

### Logging + --pdb: Best of Both Worlds

```python
# test_logging_with_pdb.py
import logging
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_payment_with_logging_and_pdb():
    """Test combining logging with pdb debugging."""
    logger = logging.getLogger(__name__)
    logger.setLevel(logging.DEBUG)
    
    # Add console handler for immediate feedback
    handler = logging.StreamHandler()
    handler.setLevel(logging.DEBUG)
    formatter = logging.Formatter('%(levelname)s - %(message)s')
    handler.setFormatter(formatter)
    logger.addHandler(handler)
    
    logger.info("=== Test starting ===")
    
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    logger.info("Processing payment")
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    logger.info("Payment processed: %s", transaction_id)
    
    # If you run with --pdb, you can inspect logs in the debugger
    transaction = processor.get_transaction(transaction_id)
    
    logger.debug("Transaction details: %s", transaction)
    
    # This assertion will fail, triggering --pdb
    assert transaction['fee'] == 2.90  # Will fail without pytest.approx()
```

```bash
$ pytest test_logging_with_pdb.py -s --pdb
```

**What happens**:
1. Logs print to console as test runs
2. When assertion fails, pdb starts
3. You can review the logs that were printed
4. You can inspect variables in the debugger

### Logging + caplog: Verify Logging Behavior

```python
# test_logging_verification.py
import logging
import pytest
from payment_processor import PaymentProcessor, MockAPIClient

def test_verify_logging_behavior(caplog):
    """Test that verifies logging behavior itself."""
    caplog.set_level(logging.DEBUG)
    
    api_client = MockAPIClient()
    processor = PaymentProcessor(api_client)
    
    # Clear any previous logs
    caplog.clear()
    
    transaction_id = processor.process_payment(
        amount=100.00,
        card_number='1234567890123456',
        cvv='123'
    )
    
    # Verify specific log messages were generated
    assert len(caplog.records) > 0, "No logs were generated"
    
    # Verify log levels
    debug_logs = [r for r in caplog.records if r.levelname == "DEBUG"]
    info_logs = [r for r in caplog.records if r.levelname == "INFO"]
    
    assert len(debug_logs) > 0, "No DEBUG logs generated"
    assert len(info_logs) > 0, "No INFO logs generated"
    
    # Verify specific messages
    messages = [r.message for r in caplog.records]
    assert any("Processing payment" in msg for msg in messages)
    assert any("Payment processed successfully" in msg for msg in messages)
    
    # Verify log order
    processing_idx = next(i for i, r in enumerate(caplog.records) 
                         if "Processing payment" in r.message)
    success_idx = next(i for i, r in enumerate(caplog.records) 
                      if "Payment processed successfully" in r.message)
    
    assert processing_idx < success_idx, "Logs out of order"
```

## Best Practices for Logging in Tests

### 1. Use Appropriate Log Levels

**DEBUG**: Detailed diagnostic information
```python
logger.debug("Processing payment: amount=%.2f, card=****%s", amount, card[-4:])
```

**INFO**: General informational messages
```python
logger.info("Payment processed successfully: transaction_id=%s", txn_id)
```

**WARNING**: Warning messages for unexpected but handled situations
```python
logger.warning("Transaction not found: %s", transaction_id)
```

**ERROR**: Error messages for failures
```python
logger.error("API charge failed: %s", str(e), exc_info=True)
```

### 2. Include Context in Log Messages

Bad:
```python
logger.info("Payment processed")
```

Good:
```python
logger.info("Payment processed: transaction_id=%s, amount=%.2f, fee=%.2f",
           transaction_id, amount, fee)
```

### 3. Use Structured Logging for Machine Parsing

```python
logger.info("payment_processed", extra={
    'transaction_id': transaction_id,
    'amount': amount,
    'fee': fee,
    'net_amount': net_amount
})
```

### 4. Log Entry and Exit of Important Functions

```python
def process_payment(self, amount, card_number, cvv):
    logger.debug("Entering process_payment: amount=%.2f", amount)
    try:
        # ... processing logic ...
        logger.debug("Exiting process_payment: transaction_id=%s", transaction_id)
        return transaction_id
    except Exception as e:
        logger.error("Exception in process_payment: %s", str(e), exc_info=True)
        raise
```

### 5. Don't Log Sensitive Information

Bad:
```python
logger.info("Processing payment: card=%s, cvv=%s", card_number, cvv)
```

Good:
```python
logger.info("Processing payment: card=****%s, cvv=***", card_number[-4:])
```

### 6. Use caplog for Testing Logging Behavior

```python
def test_error_logging(caplog):
    caplog.set_level(logging.ERROR)
    
    # ... code that should log an error ...
    
    assert "Expected error message" in caplog.text
    assert any(r.levelname == "ERROR" for r in caplog.records)
```

## Key Takeaways: Logging and Debugging Information

### 1. Logging Provides Permanent Debugging Infrastructure
Unlike breakpoints and --pdb, logs persist and can be reviewed later.

### 2. Use Appropriate Log Levels
DEBUG for detailed diagnostics, INFO for general flow, WARNING for unexpected situations, ERROR for failures.

### 3. Configure Logging in pytest.ini
Set up console and file logging with appropriate levels and formats.

### 4. Use caplog for Testing Logging Behavior
Verify that your code logs the right messages at the right levels.

### 5. Include Rich Context in Log Messages
Log relevant data like IDs, amounts, and state to make debugging easier.

### 6. Structured Logging Enables Analysis
Use JSON or other structured formats for machine-readable logs.

### 7. Combine Logging with Other Tools
Use logging alongside --pdb, breakpoints, and verbose modes for comprehensive debugging.

### 8. Never Log Sensitive Information
Mask or redact sensitive data like credit card numbers, passwords, and personal information.

## Conclusion: Building a Debugging Toolkit

Throughout this chapter, we've built a comprehensive debugging toolkit:

1. **Reading test output**: Understanding pytest's detailed failure messages
2. **Verbose modes**: Controlling output detail with -v, -vv, and -vvv
3. **The -x flag**: Stopping at the first failure for focused debugging
4. **The --pdb flag**: Interactive debugging at the point of failure
5. **Breakpoints**: Proactive debugging at specific code locations
6. **Logging**: Permanent debugging infrastructure for understanding test behavior

Each tool has its place:
- Use **output reading** to understand what failed
- Use **verbose modes** to see more detail
- Use **-x** to focus on one failure at a time
- Use **--pdb** for reactive, interactive debugging
- Use **breakpoints** for proactive, planned debugging
- Use **logging** for permanent, reviewable debugging information

Master these tools, and you'll transform debugging from frustration into systematic investigation.
