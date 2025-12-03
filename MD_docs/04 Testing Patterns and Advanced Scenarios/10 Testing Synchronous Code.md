# Chapter 10: Testing Synchronous Code

## Testing Functions and Methods

## Testing Functions and Methods

Before we explore the complexities of asynchronous code, external dependencies, or advanced mocking patterns, we must master the foundation: testing synchronous functions and methods. This is where most of your testing work happens, and where solid patterns pay dividends for years.

In this chapter, we'll build a complete testing strategy for a realistic payment processing system. We'll start with simple functions, encounter real failure modes, and progressively refine our approach until we have production-ready tests that catch bugs before they reach users.

### The Reference Implementation: A Payment Processor

Our anchor example will be a payment processing module that validates credit cards, calculates fees, and processes transactions. This is substantial enough to demonstrate real-world testing challenges while remaining focused on synchronous code patterns.

Here's our initial implementation:

```python
# payment_processor.py

def validate_credit_card(card_number: str) -> bool:
    """Validate credit card number using Luhn algorithm."""
    if not card_number.isdigit():
        return False
    
    if len(card_number) < 13 or len(card_number) > 19:
        return False
    
    # Luhn algorithm
    digits = [int(d) for d in card_number]
    checksum = 0
    
    for i in range(len(digits) - 2, -1, -2):
        doubled = digits[i] * 2
        checksum += doubled if doubled < 10 else doubled - 9
    
    for i in range(len(digits) - 1, -1, -2):
        checksum += digits[i]
    
    return checksum % 10 == 0

def calculate_processing_fee(amount: float, card_type: str) -> float:
    """Calculate processing fee based on amount and card type."""
    base_fee = amount * 0.029  # 2.9% base rate
    
    if card_type == "amex":
        base_fee += amount * 0.005  # Additional 0.5% for Amex
    
    transaction_fee = 0.30  # Fixed $0.30 per transaction
    
    return base_fee + transaction_fee

def process_payment(card_number: str, amount: float, card_type: str) -> dict:
    """Process a payment transaction."""
    if not validate_credit_card(card_number):
        return {
            "success": False,
            "error": "Invalid credit card number"
        }
    
    if amount <= 0:
        return {
            "success": False,
            "error": "Amount must be positive"
        }
    
    fee = calculate_processing_fee(amount, card_type)
    total = amount + fee
    
    return {
        "success": True,
        "amount": amount,
        "fee": fee,
        "total": total,
        "card_last_four": card_number[-4:]
    }
```

This is our starting point. It works, but we have no tests. Let's begin testing it and discover what problems emerge.

### Iteration 0: The Naive First Test

Let's write the most obvious test we can think of:

```python
# test_payment_processor.py

from payment_processor import process_payment

def test_process_payment():
    result = process_payment("4532015112830366", 100.00, "visa")
    assert result["success"] == True
```

Running this test:

```bash
$ pytest test_payment_processor.py -v
```

**Output**:

```text
test_payment_processor.py::test_process_payment PASSED                   [100%]

========================== 1 passed in 0.01s ===========================
```

Great! Our test passes. But what have we actually verified? Only that `success` is `True`. We haven't checked:

- The calculated fee
- The total amount
- The card number masking
- Error handling for invalid inputs

**Current limitation**: This test is too shallow. It passes, but it doesn't give us confidence that the payment processor actually works correctly.

### Iteration 1: Testing the Complete Success Case

Let's write a more thorough test that verifies all aspects of a successful payment:

```python
# test_payment_processor.py

from payment_processor import process_payment

def test_process_payment_success():
    result = process_payment("4532015112830366", 100.00, "visa")
    
    assert result["success"] == True
    assert result["amount"] == 100.00
    assert result["fee"] == 3.20  # 2.9% + $0.30
    assert result["total"] == 103.20
    assert result["card_last_four"] == "0366"
```

Running this test:

```bash
$ pytest test_payment_processor.py::test_process_payment_success -v
```

**Output**:

```text
test_payment_processor.py::test_process_payment_success PASSED           [100%]

========================== 1 passed in 0.01s ===========================
```

Excellent! Now we're verifying the complete behavior. But what happens when things go wrong?

**Current limitation**: We're only testing the happy path. We haven't verified error handling.

### Iteration 2: Testing Error Cases

Let's test what happens with an invalid card number:

```python
# test_payment_processor.py

from payment_processor import process_payment

def test_process_payment_success():
    result = process_payment("4532015112830366", 100.00, "visa")
    
    assert result["success"] == True
    assert result["amount"] == 100.00
    assert result["fee"] == 3.20
    assert result["total"] == 103.20
    assert result["card_last_four"] == "0366"

def test_process_payment_invalid_card():
    result = process_payment("1234567890123456", 100.00, "visa")
    
    assert result["success"] == False
    assert result["error"] == "Invalid credit card number"
```

Running this test:

```bash
$ pytest test_payment_processor.py::test_process_payment_invalid_card -v
```

**Output**:

```text
test_payment_processor.py::test_process_payment_invalid_card PASSED      [100%]

========================== 1 passed in 0.01s ===========================
```

Good! But now we have a problem: we're repeating the card number "4532015112830366" in multiple tests. What if we need to change it? What if we want to test with different valid cards?

**Current limitation**: Test data is hardcoded and duplicated across tests.

### Iteration 3: Extracting Test Data

Let's use module-level constants for our test data:

```python
# test_payment_processor.py

from payment_processor import process_payment

# Test data
VALID_VISA = "4532015112830366"
INVALID_CARD = "1234567890123456"

def test_process_payment_success():
    result = process_payment(VALID_VISA, 100.00, "visa")
    
    assert result["success"] == True
    assert result["amount"] == 100.00
    assert result["fee"] == 3.20
    assert result["total"] == 103.20
    assert result["card_last_four"] == "0366"

def test_process_payment_invalid_card():
    result = process_payment(INVALID_CARD, 100.00, "visa")
    
    assert result["success"] == False
    assert result["error"] == "Invalid credit card number"

def test_process_payment_negative_amount():
    result = process_payment(VALID_VISA, -50.00, "visa")
    
    assert result["success"] == False
    assert result["error"] == "Amount must be positive"
```

Running all tests:

```bash
$ pytest test_payment_processor.py -v
```

**Output**:

```text
test_payment_processor.py::test_process_payment_success PASSED           [ 33%]
test_payment_processor.py::test_process_payment_invalid_card PASSED      [ 66%]
test_payment_processor.py::test_process_payment_negative_amount PASSED   [100%]

========================== 3 passed in 0.02s ===========================
```

Better! Our test data is now centralized. But look at the repetition in our assertions. Every test follows the same pattern: call the function, assert multiple fields. Can we make this cleaner?

**Current limitation**: Repetitive assertion patterns make tests verbose and harder to maintain.

### Iteration 4: Helper Functions for Common Assertions

Let's create helper functions to reduce repetition:

```python
# test_payment_processor.py

from payment_processor import process_payment

# Test data
VALID_VISA = "4532015112830366"
INVALID_CARD = "1234567890123456"

def assert_successful_payment(result, expected_amount, expected_fee, expected_last_four):
    """Helper to verify successful payment results."""
    assert result["success"] == True
    assert result["amount"] == expected_amount
    assert result["fee"] == expected_fee
    assert result["total"] == expected_amount + expected_fee
    assert result["card_last_four"] == expected_last_four

def assert_failed_payment(result, expected_error):
    """Helper to verify failed payment results."""
    assert result["success"] == False
    assert result["error"] == expected_error

def test_process_payment_success():
    result = process_payment(VALID_VISA, 100.00, "visa")
    assert_successful_payment(result, 100.00, 3.20, "0366")

def test_process_payment_invalid_card():
    result = process_payment(INVALID_CARD, 100.00, "visa")
    assert_failed_payment(result, "Invalid credit card number")

def test_process_payment_negative_amount():
    result = process_payment(VALID_VISA, -50.00, "visa")
    assert_failed_payment(result, "Amount must be positive")
```

Running all tests:

```bash
$ pytest test_payment_processor.py -v
```

**Output**:

```text
test_payment_processor.py::test_process_payment_success PASSED           [ 33%]
test_payment_processor.py::test_process_payment_invalid_card PASSED      [ 66%]
test_payment_processor.py::test_process_payment_negative_amount PASSED   [100%]

========================== 3 passed in 0.02s ===========================
```

Much cleaner! Our tests are now more readable and maintainable. But we still have a problem: we're only testing one card type (Visa). What about American Express, which has different fees?

**Current limitation**: We're not testing the card type fee variations.

### Iteration 5: Testing Different Card Types

Let's add tests for American Express:

```python
# test_payment_processor.py

from payment_processor import process_payment

# Test data
VALID_VISA = "4532015112830366"
VALID_AMEX = "378282246310005"
INVALID_CARD = "1234567890123456"

def assert_successful_payment(result, expected_amount, expected_fee, expected_last_four):
    """Helper to verify successful payment results."""
    assert result["success"] == True
    assert result["amount"] == expected_amount
    assert result["fee"] == expected_fee
    assert result["total"] == expected_amount + expected_fee
    assert result["card_last_four"] == expected_last_four

def assert_failed_payment(result, expected_error):
    """Helper to verify failed payment results."""
    assert result["success"] == False
    assert result["error"] == expected_error

def test_process_payment_visa():
    result = process_payment(VALID_VISA, 100.00, "visa")
    # Visa: 2.9% + $0.30 = $2.90 + $0.30 = $3.20
    assert_successful_payment(result, 100.00, 3.20, "0366")

def test_process_payment_amex():
    result = process_payment(VALID_AMEX, 100.00, "amex")
    # Amex: 3.4% + $0.30 = $3.40 + $0.30 = $3.70
    assert_successful_payment(result, 100.00, 3.70, "0005")

def test_process_payment_invalid_card():
    result = process_payment(INVALID_CARD, 100.00, "visa")
    assert_failed_payment(result, "Invalid credit card number")

def test_process_payment_negative_amount():
    result = process_payment(VALID_VISA, -50.00, "visa")
    assert_failed_payment(result, "Amount must be positive")
```

Running all tests:

```bash
$ pytest test_payment_processor.py -v
```

**Output**:

```text
test_payment_processor.py::test_process_payment_visa PASSED              [ 25%]
test_payment_processor.py::test_process_payment_amex PASSED              [ 50%]
test_payment_processor.py::test_process_payment_invalid_card PASSED      [ 75%]
test_payment_processor.py::test_process_payment_negative_amount PASSED   [100%]

========================== 4 passed in 0.02s ===========================
```

Excellent! Now we're testing both card types. But wait—let's verify that our fee calculation is actually correct. Let me run the Amex test again with verbose output to see the actual values:

```bash
$ pytest test_payment_processor.py::test_process_payment_amex -v
```

The test passes, which means our fee calculation is working. But there's a subtle issue here: we're calculating the expected fee in our heads and hardcoding it. What if we make a mistake? What if the fee structure changes?

**Current limitation**: Fee calculations are hardcoded in tests, making them brittle and error-prone.

### Diagnostic Analysis: When Tests Pass But Shouldn't

Let's intentionally introduce a bug to see what happens. Suppose we accidentally change the Amex fee calculation:

```python
# payment_processor.py (with bug)

def calculate_processing_fee(amount: float, card_type: str) -> float:
    """Calculate processing fee based on amount and card type."""
    base_fee = amount * 0.029  # 2.9% base rate
    
    if card_type == "amex":
        base_fee += amount * 0.010  # BUG: Should be 0.005, not 0.010
    
    transaction_fee = 0.30
    
    return base_fee + transaction_fee
```

Now let's run our test:

```bash
$ pytest test_payment_processor.py::test_process_payment_amex -v
```

**Output**:

```text
test_payment_processor.py::test_process_payment_amex FAILED              [100%]

================================= FAILURES =================================
_______________________ test_process_payment_amex __________________________

    def test_process_payment_amex():
        result = process_payment(VALID_AMEX, 100.00, "amex")
>       assert_successful_payment(result, 100.00, 3.70, "0005")

test_payment_processor.py:30: 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _

result = {'success': True, 'amount': 100.0, 'fee': 4.2, 'total': 104.2, ...}
expected_amount = 100.0, expected_fee = 3.7, expected_last_four = '0005'

    def assert_successful_payment(result, expected_amount, expected_fee, expected_last_four):
        """Helper to verify successful payment results."""
        assert result["success"] == True
        assert result["amount"] == expected_amount
>       assert result["fee"] == expected_fee
E       AssertionError: assert 4.2 == 3.7
E        +  where 4.2 = {'success': True, 'amount': 100.0, 'fee': 4.2, 'total': 104.2, 'card_last_four': '0005'}['fee']

test_payment_processor.py:16: AssertionError
========================== 1 failed in 0.03s ===========================
```

### Reading the Failure

**The summary line**: `FAILED test_payment_processor.py::test_process_payment_amex - AssertionError`
- What this tells us: The test failed due to an assertion error, not an exception in the code

**The traceback**:
```
    def assert_successful_payment(result, expected_amount, expected_fee, expected_last_four):
        """Helper to verify successful payment results."""
        assert result["success"] == True
        assert result["amount"] == expected_amount
>       assert result["fee"] == expected_fee
E       AssertionError: assert 4.2 == 3.7
```
- What this tells us: The failure occurred in our helper function at the fee assertion
- Key line: `assert 4.2 == 3.7` - The actual fee is $4.20, but we expected $3.70

**The assertion introspection**:
```
E       AssertionError: assert 4.2 == 3.7
E        +  where 4.2 = {'success': True, 'amount': 100.0, 'fee': 4.2, 'total': 104.2, 'card_last_four': '0005'}['fee']
```
- What this tells us: Pytest shows us the complete result dictionary, revealing that the fee is $4.20 instead of $3.70

**Root cause identified**: The Amex fee calculation is incorrect—it's charging 1% extra instead of 0.5% extra.

**Why the current approach caught this**: Our hardcoded expected value (3.70) acted as a specification. When the implementation deviated, the test failed.

Good! Our test caught the bug. Let's fix it:

```python
# payment_processor.py (fixed)

def calculate_processing_fee(amount: float, card_type: str) -> float:
    """Calculate processing fee based on amount and card type."""
    base_fee = amount * 0.029  # 2.9% base rate
    
    if card_type == "amex":
        base_fee += amount * 0.005  # Fixed: 0.5% for Amex
    
    transaction_fee = 0.30
    
    return base_fee + transaction_fee
```

Now all tests pass again. But this raises an important question: should we calculate the expected fee in our tests, or hardcode it?

### When to Apply This Solution

**What it optimizes for**:
- Clarity: Hardcoded values make expectations explicit
- Specification: Tests document the exact behavior
- Bug detection: Changes to calculations are immediately visible

**What it sacrifices**:
- Maintainability: If fee structure changes, all tests need updates
- Duplication: Fee calculation logic exists in both code and tests

**When to choose this approach**:
- Testing business logic with specific requirements
- When the calculation is the core behavior being tested
- When you want tests to serve as documentation

**When to avoid this approach**:
- Testing helper functions where the calculation itself isn't critical
- When the calculation is complex and likely to change
- When you're testing the calculation logic itself (use property-based testing instead)

**Current state**: We have solid tests for our payment processor. We've tested success cases, error cases, and different card types. Our tests are readable and maintainable.

### Testing Individual Functions

So far, we've been testing `process_payment()`, which is a high-level function that calls other functions. But what about testing `validate_credit_card()` and `calculate_processing_fee()` directly?

Let's add tests for these lower-level functions:

```python
# test_payment_processor.py

from payment_processor import (
    process_payment,
    validate_credit_card,
    calculate_processing_fee
)

# Test data
VALID_VISA = "4532015112830366"
VALID_AMEX = "378282246310005"
INVALID_CARD = "1234567890123456"

# Helper functions
def assert_successful_payment(result, expected_amount, expected_fee, expected_last_four):
    """Helper to verify successful payment results."""
    assert result["success"] == True
    assert result["amount"] == expected_amount
    assert result["fee"] == expected_fee
    assert result["total"] == expected_amount + expected_fee
    assert result["card_last_four"] == expected_last_four

def assert_failed_payment(result, expected_error):
    """Helper to verify failed payment results."""
    assert result["success"] == False
    assert result["error"] == expected_error

# Tests for validate_credit_card()
def test_validate_credit_card_valid_visa():
    assert validate_credit_card(VALID_VISA) == True

def test_validate_credit_card_valid_amex():
    assert validate_credit_card(VALID_AMEX) == True

def test_validate_credit_card_invalid_luhn():
    assert validate_credit_card(INVALID_CARD) == False

def test_validate_credit_card_non_numeric():
    assert validate_credit_card("1234-5678-9012-3456") == False

def test_validate_credit_card_too_short():
    assert validate_credit_card("123456789012") == False

def test_validate_credit_card_too_long():
    assert validate_credit_card("12345678901234567890") == False

# Tests for calculate_processing_fee()
def test_calculate_processing_fee_visa():
    fee = calculate_processing_fee(100.00, "visa")
    assert fee == 3.20

def test_calculate_processing_fee_amex():
    fee = calculate_processing_fee(100.00, "amex")
    assert fee == 3.70

def test_calculate_processing_fee_small_amount():
    fee = calculate_processing_fee(1.00, "visa")
    assert fee == 0.33  # $0.029 + $0.30 = $0.329, rounded

# Tests for process_payment()
def test_process_payment_visa():
    result = process_payment(VALID_VISA, 100.00, "visa")
    assert_successful_payment(result, 100.00, 3.20, "0366")

def test_process_payment_amex():
    result = process_payment(VALID_AMEX, 100.00, "amex")
    assert_successful_payment(result, 100.00, 3.70, "0005")

def test_process_payment_invalid_card():
    result = process_payment(INVALID_CARD, 100.00, "visa")
    assert_failed_payment(result, "Invalid credit card number")

def test_process_payment_negative_amount():
    result = process_payment(VALID_VISA, -50.00, "visa")
    assert_failed_payment(result, "Amount must be positive")
```

Running all tests:

```bash
$ pytest test_payment_processor.py -v
```

**Output**:

```text
test_payment_processor.py::test_validate_credit_card_valid_visa PASSED   [  7%]
test_payment_processor.py::test_validate_credit_card_valid_amex PASSED   [ 14%]
test_payment_processor.py::test_validate_credit_card_invalid_luhn PASSED [ 21%]
test_payment_processor.py::test_validate_credit_card_non_numeric PASSED  [ 28%]
test_payment_processor.py::test_validate_credit_card_too_short PASSED    [ 35%]
test_payment_processor.py::test_validate_credit_card_too_long PASSED     [ 42%]
test_payment_processor.py::test_calculate_processing_fee_visa PASSED     [ 50%]
test_payment_processor.py::test_calculate_processing_fee_amex PASSED     [ 57%]
test_payment_processor.py::test_calculate_processing_fee_small_amount FAILED [ 64%]
test_payment_processor.py::test_process_payment_visa PASSED              [ 71%]
test_payment_processor.py::test_process_payment_amex PASSED              [ 78%]
test_payment_processor.py::test_process_payment_invalid_card PASSED      [ 85%]
test_payment_processor.py::test_process_payment_negative_amount PASSED   [ 92%]

================================= FAILURES =================================
_________________ test_calculate_processing_fee_small_amount ________________

    def test_calculate_processing_fee_small_amount():
        fee = calculate_processing_fee(1.00, "visa")
>       assert fee == 0.33
E       AssertionError: assert 0.329 == 0.33

test_payment_processor.py:52: AssertionError
========================== 1 failed, 12 passed in 0.04s =======================
```

### Diagnostic Analysis: Floating Point Precision

**The complete output**:
```
test_payment_processor.py::test_calculate_processing_fee_small_amount FAILED [ 64%]

================================= FAILURES =================================
_________________ test_calculate_processing_fee_small_amount ________________

    def test_calculate_processing_fee_small_amount():
        fee = calculate_processing_fee(1.00, "visa")
>       assert fee == 0.33
E       AssertionError: assert 0.329 == 0.33

test_payment_processor.py:52: AssertionError
```

**Let's parse this section by section**:

1. **The summary line**: `FAILED test_payment_processor.py::test_calculate_processing_fee_small_amount - AssertionError`
   - What this tells us: The test failed on an assertion, not an exception

2. **The traceback**:
```
    def test_calculate_processing_fee_small_amount():
        fee = calculate_processing_fee(1.00, "visa")
>       assert fee == 0.33
E       AssertionError: assert 0.329 == 0.33
```
- What this tells us: We expected 0.33, but got 0.329
- Key line: The actual value is 0.329, which is the mathematically correct result ($1.00 * 0.029 + $0.30 = $0.329)

3. **The assertion introspection**:
```
E       AssertionError: assert 0.329 == 0.33
```
- What this tells us: This is a floating-point comparison issue

**Root cause identified**: We're comparing floating-point numbers with exact equality, which is problematic. The actual fee is $0.329, which we incorrectly rounded to $0.33 in our test.

**Why the current approach can't solve this**: Exact equality (`==`) for floating-point numbers is fragile. We need approximate comparison.

**What we need**: Pytest's `approx()` function for floating-point comparisons.

Let's fix this test:

```python
# test_payment_processor.py

from pytest import approx
from payment_processor import (
    process_payment,
    validate_credit_card,
    calculate_processing_fee
)

# ... (previous code remains the same)

def test_calculate_processing_fee_small_amount():
    fee = calculate_processing_fee(1.00, "visa")
    assert fee == approx(0.329)  # Use approx for floating-point comparison
```

Running the test again:

```bash
$ pytest test_payment_processor.py::test_calculate_processing_fee_small_amount -v
```

**Output**:

```text
test_payment_processor.py::test_calculate_processing_fee_small_amount PASSED [100%]

========================== 1 passed in 0.01s ===========================
```

Perfect! Now all our tests pass. Let's run the complete test suite:

```bash
$ pytest test_payment_processor.py -v
```

**Output**:

```text
test_payment_processor.py::test_validate_credit_card_valid_visa PASSED   [  7%]
test_payment_processor.py::test_validate_credit_card_valid_amex PASSED   [ 14%]
test_payment_processor.py::test_validate_credit_card_invalid_luhn PASSED [ 21%]
test_payment_processor.py::test_validate_credit_card_non_numeric PASSED  [ 28%]
test_payment_processor.py::test_validate_credit_card_too_short PASSED    [ 35%]
test_payment_processor.py::test_validate_credit_card_too_long PASSED     [ 42%]
test_payment_processor.py::test_calculate_processing_fee_visa PASSED     [ 50%]
test_payment_processor.py::test_calculate_processing_fee_amex PASSED     [ 57%]
test_payment_processor.py::test_calculate_processing_fee_small_amount PASSED [ 64%]
test_payment_processor.py::test_process_payment_visa PASSED              [ 71%]
test_payment_processor.py::test_process_payment_amex PASSED              [ 78%]
test_payment_processor.py::test_process_payment_invalid_card PASSED      [ 85%]
test_payment_processor.py::test_process_payment_negative_amount PASSED   [ 92%]

========================== 13 passed in 0.03s ==========================
```

### The Journey: From Problem to Solution

| Iteration | Problem                          | Solution Applied                | Result                    |
|-----------|----------------------------------|---------------------------------|---------------------------|
| 0         | No tests                         | Write first test                | Basic coverage            |
| 1         | Shallow assertions               | Test all return fields          | Complete verification     |
| 2         | Only happy path tested           | Add error case tests            | Error handling verified   |
| 3         | Hardcoded test data              | Extract constants               | Better maintainability    |
| 4         | Repetitive assertions            | Create helper functions         | Cleaner tests             |
| 5         | Single card type tested          | Add Amex tests                  | Multiple scenarios        |
| 6         | Floating-point comparison issues | Use pytest.approx()             | Robust numeric tests      |
| 7         | Only high-level function tested  | Test individual functions       | Comprehensive coverage    |

### Final Implementation

Here's our complete test suite:

```python
# test_payment_processor.py

from pytest import approx
from payment_processor import (
    process_payment,
    validate_credit_card,
    calculate_processing_fee
)

# Test data
VALID_VISA = "4532015112830366"
VALID_AMEX = "378282246310005"
INVALID_CARD = "1234567890123456"

# Helper functions
def assert_successful_payment(result, expected_amount, expected_fee, expected_last_four):
    """Helper to verify successful payment results."""
    assert result["success"] == True
    assert result["amount"] == expected_amount
    assert result["fee"] == approx(expected_fee)
    assert result["total"] == approx(expected_amount + expected_fee)
    assert result["card_last_four"] == expected_last_four

def assert_failed_payment(result, expected_error):
    """Helper to verify failed payment results."""
    assert result["success"] == False
    assert result["error"] == expected_error

# Tests for validate_credit_card()
def test_validate_credit_card_valid_visa():
    assert validate_credit_card(VALID_VISA) == True

def test_validate_credit_card_valid_amex():
    assert validate_credit_card(VALID_AMEX) == True

def test_validate_credit_card_invalid_luhn():
    assert validate_credit_card(INVALID_CARD) == False

def test_validate_credit_card_non_numeric():
    assert validate_credit_card("1234-5678-9012-3456") == False

def test_validate_credit_card_too_short():
    assert validate_credit_card("123456789012") == False

def test_validate_credit_card_too_long():
    assert validate_credit_card("12345678901234567890") == False

# Tests for calculate_processing_fee()
def test_calculate_processing_fee_visa():
    fee = calculate_processing_fee(100.00, "visa")
    assert fee == approx(3.20)

def test_calculate_processing_fee_amex():
    fee = calculate_processing_fee(100.00, "amex")
    assert fee == approx(3.70)

def test_calculate_processing_fee_small_amount():
    fee = calculate_processing_fee(1.00, "visa")
    assert fee == approx(0.329)

# Tests for process_payment()
def test_process_payment_visa():
    result = process_payment(VALID_VISA, 100.00, "visa")
    assert_successful_payment(result, 100.00, 3.20, "0366")

def test_process_payment_amex():
    result = process_payment(VALID_AMEX, 100.00, "amex")
    assert_successful_payment(result, 100.00, 3.70, "0005")

def test_process_payment_invalid_card():
    result = process_payment(INVALID_CARD, 100.00, "visa")
    assert_failed_payment(result, "Invalid credit card number")

def test_process_payment_negative_amount():
    result = process_payment(VALID_VISA, -50.00, "visa")
    assert_failed_payment(result, "Amount must be positive")
```

### Decision Framework: Testing Granularity

**When to test individual functions**:
- The function has complex logic worth testing in isolation
- The function is reused in multiple places
- You want to test edge cases that are hard to trigger through high-level functions
- The function's behavior is independent of other functions

**When to test only high-level functions**:
- Individual functions are simple and unlikely to fail independently
- The integration between functions is the primary concern
- Testing individual functions would duplicate coverage
- The functions are private implementation details

**For our payment processor**:
- We test `validate_credit_card()` individually because it has complex logic (Luhn algorithm)
- We test `calculate_processing_fee()` individually because it has business logic worth documenting
- We test `process_payment()` to verify the integration of all components

### Lessons Learned

1. **Start simple, then refine**: Begin with basic tests and progressively add coverage
2. **Test both success and failure paths**: Don't just test the happy path
3. **Use helper functions to reduce repetition**: Extract common assertion patterns
4. **Use pytest.approx() for floating-point comparisons**: Never use `==` for floats
5. **Test at multiple levels**: Test both individual functions and their integration
6. **Make test data explicit**: Use constants for test data to improve maintainability
7. **Let failures guide you**: Each failure reveals what needs to be tested next

## Testing Classes and Object-Oriented Code

## Testing Classes and Object-Oriented Code

Functions are the foundation, but most real-world Python code is organized into classes. Testing classes introduces new challenges: managing object state, testing methods that interact with each other, and handling inheritance hierarchies.

Let's extend our payment processor into a class-based design and discover how to test it effectively.

### The Reference Implementation: A Payment Gateway Class

We'll refactor our payment processor into a `PaymentGateway` class that maintains state and provides a more realistic API:

```python
# payment_gateway.py

class PaymentGateway:
    """A payment gateway that processes credit card transactions."""
    
    def __init__(self, merchant_id: str, api_key: str):
        self.merchant_id = merchant_id
        self.api_key = api_key
        self.transaction_history = []
        self.total_processed = 0.0
    
    def validate_credit_card(self, card_number: str) -> bool:
        """Validate credit card number using Luhn algorithm."""
        if not card_number.isdigit():
            return False
        
        if len(card_number) < 13 or len(card_number) > 19:
            return False
        
        # Luhn algorithm
        digits = [int(d) for d in card_number]
        checksum = 0
        
        for i in range(len(digits) - 2, -1, -2):
            doubled = digits[i] * 2
            checksum += doubled if doubled < 10 else doubled - 9
        
        for i in range(len(digits) - 1, -1, -2):
            checksum += digits[i]
        
        return checksum % 10 == 0
    
    def calculate_fee(self, amount: float, card_type: str) -> float:
        """Calculate processing fee based on amount and card type."""
        base_fee = amount * 0.029
        
        if card_type == "amex":
            base_fee += amount * 0.005
        
        return base_fee + 0.30
    
    def process_payment(self, card_number: str, amount: float, card_type: str) -> dict:
        """Process a payment transaction."""
        if not self.validate_credit_card(card_number):
            return {
                "success": False,
                "error": "Invalid credit card number"
            }
        
        if amount <= 0:
            return {
                "success": False,
                "error": "Amount must be positive"
            }
        
        fee = self.calculate_fee(amount, card_type)
        total = amount + fee
        
        # Record transaction
        transaction = {
            "card_last_four": card_number[-4:],
            "amount": amount,
            "fee": fee,
            "total": total
        }
        self.transaction_history.append(transaction)
        self.total_processed += total
        
        return {
            "success": True,
            "amount": amount,
            "fee": fee,
            "total": total,
            "card_last_four": card_number[-4:]
        }
    
    def get_transaction_count(self) -> int:
        """Get the number of processed transactions."""
        return len(self.transaction_history)
    
    def get_total_processed(self) -> float:
        """Get the total amount processed."""
        return self.total_processed
```

This is our new reference implementation. It's more realistic: it maintains state, tracks transaction history, and provides methods to query that state.

### Iteration 0: The Naive First Test

Let's write the most obvious test:

```python
# test_payment_gateway.py

from payment_gateway import PaymentGateway

def test_process_payment():
    gateway = PaymentGateway("MERCHANT123", "secret_key")
    result = gateway.process_payment("4532015112830366", 100.00, "visa")
    assert result["success"] == True
```

Running this test:

```bash
$ pytest test_payment_gateway.py::test_process_payment -v
```

**Output**:

```text
test_payment_gateway.py::test_process_payment PASSED                     [100%]

========================== 1 passed in 0.01s ===========================
```

Good! But we have a problem: we're creating a new `PaymentGateway` instance in every test. This is repetitive and makes our tests harder to maintain.

**Current limitation**: Gateway instantiation is duplicated across tests.

### Iteration 1: Using a Fixture for Setup

Let's create a fixture to provide a gateway instance:

```python
# test_payment_gateway.py

import pytest
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

def test_process_payment(gateway):
    result = gateway.process_payment("4532015112830366", 100.00, "visa")
    assert result["success"] == True
```

Running this test:

```bash
$ pytest test_payment_gateway.py::test_process_payment -v
```

**Output**:

```text
test_payment_gateway.py::test_process_payment PASSED                     [100%]

========================== 1 passed in 0.01s ===========================
```

Much better! Now we can reuse the gateway fixture across multiple tests. But there's a subtle issue: what happens if one test modifies the gateway's state? Will it affect other tests?

**Current limitation**: We don't know if our fixture provides test isolation.

### Iteration 2: Testing State Isolation

Let's write two tests that modify the gateway's state:

```python
# test_payment_gateway.py

import pytest
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

def test_first_transaction(gateway):
    gateway.process_payment("4532015112830366", 100.00, "visa")
    assert gateway.get_transaction_count() == 1

def test_second_transaction(gateway):
    # This test should start with a clean gateway
    assert gateway.get_transaction_count() == 0
    gateway.process_payment("4532015112830366", 50.00, "visa")
    assert gateway.get_transaction_count() == 1
```

Running these tests:

```bash
$ pytest test_payment_gateway.py -v
```

**Output**:

```text
test_payment_gateway.py::test_first_transaction PASSED                   [ 50%]
test_payment_gateway.py::test_second_transaction PASSED                  [100%]

========================== 2 passed in 0.01s ===========================
```

Excellent! Both tests pass, which means each test gets a fresh gateway instance. This is because pytest fixtures have **function scope** by default—they're recreated for each test function.

**Verification**: Our fixture provides proper test isolation. Each test starts with a clean gateway.

### Iteration 3: Testing State Accumulation

Now let's test that the gateway correctly accumulates state across multiple transactions:

```python
# test_payment_gateway.py

import pytest
from pytest import approx
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

def test_transaction_count_accumulates(gateway):
    assert gateway.get_transaction_count() == 0
    
    gateway.process_payment("4532015112830366", 100.00, "visa")
    assert gateway.get_transaction_count() == 1
    
    gateway.process_payment("4532015112830366", 50.00, "visa")
    assert gateway.get_transaction_count() == 2
    
    gateway.process_payment("4532015112830366", 25.00, "visa")
    assert gateway.get_transaction_count() == 3

def test_total_processed_accumulates(gateway):
    assert gateway.get_total_processed() == approx(0.0)
    
    # First transaction: $100 + $3.20 fee = $103.20
    gateway.process_payment("4532015112830366", 100.00, "visa")
    assert gateway.get_total_processed() == approx(103.20)
    
    # Second transaction: $50 + $1.75 fee = $51.75
    gateway.process_payment("4532015112830366", 50.00, "visa")
    assert gateway.get_total_processed() == approx(103.20 + 51.75)
```

Running these tests:

```bash
$ pytest test_payment_gateway.py::test_transaction_count_accumulates -v
$ pytest test_payment_gateway.py::test_total_processed_accumulates -v
```

**Output**:

```text
test_payment_gateway.py::test_transaction_count_accumulates PASSED       [100%]

========================== 1 passed in 0.01s ===========================

test_payment_gateway.py::test_total_processed_accumulates PASSED         [100%]

========================== 1 passed in 0.01s ===========================
```

Perfect! Our tests verify that the gateway correctly maintains state across multiple transactions within a single test.

**Current state**: We can test both state isolation (between tests) and state accumulation (within tests).

### Iteration 4: Testing Method Interactions

Our gateway has methods that depend on each other. Let's test that they interact correctly:

```python
# test_payment_gateway.py

import pytest
from pytest import approx
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

def test_failed_transaction_not_recorded(gateway):
    """Failed transactions should not be added to history."""
    # Try to process with invalid card
    result = gateway.process_payment("1234567890123456", 100.00, "visa")
    
    assert result["success"] == False
    assert gateway.get_transaction_count() == 0
    assert gateway.get_total_processed() == approx(0.0)

def test_successful_transaction_recorded(gateway):
    """Successful transactions should be added to history."""
    result = gateway.process_payment("4532015112830366", 100.00, "visa")
    
    assert result["success"] == True
    assert gateway.get_transaction_count() == 1
    assert gateway.get_total_processed() == approx(103.20)
```

Running these tests:

```bash
$ pytest test_payment_gateway.py::test_failed_transaction_not_recorded -v
$ pytest test_payment_gateway.py::test_successful_transaction_recorded -v
```

**Output**:

```text
test_payment_gateway.py::test_failed_transaction_not_recorded PASSED     [100%]

========================== 1 passed in 0.01s ===========================

test_payment_gateway.py::test_successful_transaction_recorded PASSED     [100%]

========================== 1 passed in 0.01s ===========================
```

Excellent! We're now testing the interaction between `process_payment()` and the state-tracking methods.

**Current limitation**: We're testing the gateway's public interface, but what about testing individual methods in isolation?

### Iteration 5: Testing Individual Methods

Let's add tests for the individual methods:

```python
# test_payment_gateway.py

import pytest
from pytest import approx
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

# Tests for validate_credit_card()
def test_validate_credit_card_valid(gateway):
    assert gateway.validate_credit_card("4532015112830366") == True

def test_validate_credit_card_invalid(gateway):
    assert gateway.validate_credit_card("1234567890123456") == False

def test_validate_credit_card_non_numeric(gateway):
    assert gateway.validate_credit_card("1234-5678-9012-3456") == False

# Tests for calculate_fee()
def test_calculate_fee_visa(gateway):
    fee = gateway.calculate_fee(100.00, "visa")
    assert fee == approx(3.20)

def test_calculate_fee_amex(gateway):
    fee = gateway.calculate_fee(100.00, "amex")
    assert fee == approx(3.70)

# Tests for process_payment()
def test_process_payment_success(gateway):
    result = gateway.process_payment("4532015112830366", 100.00, "visa")
    assert result["success"] == True
    assert result["amount"] == approx(100.00)
    assert result["fee"] == approx(3.20)

def test_process_payment_invalid_card(gateway):
    result = gateway.process_payment("1234567890123456", 100.00, "visa")
    assert result["success"] == False
    assert result["error"] == "Invalid credit card number"

# Tests for state management
def test_transaction_count_accumulates(gateway):
    assert gateway.get_transaction_count() == 0
    gateway.process_payment("4532015112830366", 100.00, "visa")
    assert gateway.get_transaction_count() == 1
    gateway.process_payment("4532015112830366", 50.00, "visa")
    assert gateway.get_transaction_count() == 2

def test_failed_transaction_not_recorded(gateway):
    result = gateway.process_payment("1234567890123456", 100.00, "visa")
    assert result["success"] == False
    assert gateway.get_transaction_count() == 0
```

Running all tests:

```bash
$ pytest test_payment_gateway.py -v
```

**Output**:

```text
test_payment_gateway.py::test_validate_credit_card_valid PASSED          [ 10%]
test_payment_gateway.py::test_validate_credit_card_invalid PASSED        [ 20%]
test_payment_gateway.py::test_validate_credit_card_non_numeric PASSED    [ 30%]
test_payment_gateway.py::test_calculate_fee_visa PASSED                  [ 40%]
test_payment_gateway.py::test_calculate_fee_amex PASSED                  [ 50%]
test_payment_gateway.py::test_process_payment_success PASSED             [ 60%]
test_payment_gateway.py::test_process_payment_invalid_card PASSED        [ 70%]
test_payment_gateway.py::test_transaction_count_accumulates PASSED       [ 80%]
test_payment_gateway.py::test_failed_transaction_not_recorded PASSED     [ 90%]

========================== 9 passed in 0.03s ==========================
```

Perfect! We now have comprehensive coverage of our `PaymentGateway` class.

### Testing Class Initialization

We should also test that the gateway is initialized correctly:

```python
# test_payment_gateway.py

import pytest
from pytest import approx
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

def test_gateway_initialization():
    """Test that gateway is initialized with correct state."""
    gateway = PaymentGateway("MERCHANT123", "secret_key")
    
    assert gateway.merchant_id == "MERCHANT123"
    assert gateway.api_key == "secret_key"
    assert gateway.transaction_history == []
    assert gateway.total_processed == 0.0

# ... (rest of the tests remain the same)
```

Running this test:

```bash
$ pytest test_payment_gateway.py::test_gateway_initialization -v
```

**Output**:

```text
test_payment_gateway.py::test_gateway_initialization PASSED              [100%]

========================== 1 passed in 0.01s ===========================
```

### Testing Class Inheritance

Let's extend our gateway with a subclass that adds fraud detection:

```python
# payment_gateway.py

class PaymentGateway:
    """A payment gateway that processes credit card transactions."""
    
    def __init__(self, merchant_id: str, api_key: str):
        self.merchant_id = merchant_id
        self.api_key = api_key
        self.transaction_history = []
        self.total_processed = 0.0
    
    # ... (previous methods remain the same)

class FraudDetectionGateway(PaymentGateway):
    """A payment gateway with fraud detection capabilities."""
    
    def __init__(self, merchant_id: str, api_key: str, fraud_threshold: float = 1000.0):
        super().__init__(merchant_id, api_key)
        self.fraud_threshold = fraud_threshold
        self.flagged_transactions = []
    
    def is_suspicious(self, amount: float, card_number: str) -> bool:
        """Check if a transaction is suspicious."""
        # Flag transactions over threshold
        if amount > self.fraud_threshold:
            return True
        
        # Flag if same card used multiple times in short period
        recent_cards = [t["card_last_four"] for t in self.transaction_history[-5:]]
        if recent_cards.count(card_number[-4:]) >= 3:
            return True
        
        return False
    
    def process_payment(self, card_number: str, amount: float, card_type: str) -> dict:
        """Process a payment with fraud detection."""
        # Check for fraud before processing
        if self.is_suspicious(amount, card_number):
            transaction = {
                "card_last_four": card_number[-4:],
                "amount": amount,
                "reason": "Suspicious activity detected"
            }
            self.flagged_transactions.append(transaction)
            
            return {
                "success": False,
                "error": "Transaction flagged for review"
            }
        
        # Process normally if not suspicious
        return super().process_payment(card_number, amount, card_type)
```

Now let's test the subclass:

```python
# test_payment_gateway.py

import pytest
from pytest import approx
from payment_gateway import PaymentGateway, FraudDetectionGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

@pytest.fixture
def fraud_gateway():
    """Provide a FraudDetectionGateway instance for testing."""
    return FraudDetectionGateway("MERCHANT123", "secret_key", fraud_threshold=500.0)

# ... (previous tests remain the same)

# Tests for FraudDetectionGateway
def test_fraud_gateway_initialization():
    """Test that fraud gateway is initialized correctly."""
    gateway = FraudDetectionGateway("MERCHANT123", "secret_key", fraud_threshold=500.0)
    
    assert gateway.merchant_id == "MERCHANT123"
    assert gateway.api_key == "secret_key"
    assert gateway.fraud_threshold == 500.0
    assert gateway.flagged_transactions == []

def test_fraud_detection_high_amount(fraud_gateway):
    """Test that high-value transactions are flagged."""
    result = fraud_gateway.process_payment("4532015112830366", 1000.00, "visa")
    
    assert result["success"] == False
    assert result["error"] == "Transaction flagged for review"
    assert len(fraud_gateway.flagged_transactions) == 1

def test_fraud_detection_normal_amount(fraud_gateway):
    """Test that normal transactions are not flagged."""
    result = fraud_gateway.process_payment("4532015112830366", 100.00, "visa")
    
    assert result["success"] == True
    assert len(fraud_gateway.flagged_transactions) == 0

def test_fraud_detection_repeated_card(fraud_gateway):
    """Test that repeated card usage is flagged."""
    card = "4532015112830366"
    
    # Process 3 transactions with same card
    for _ in range(3):
        fraud_gateway.process_payment(card, 100.00, "visa")
    
    # Fourth transaction should be flagged
    result = fraud_gateway.process_payment(card, 100.00, "visa")
    
    assert result["success"] == False
    assert result["error"] == "Transaction flagged for review"
    assert len(fraud_gateway.flagged_transactions) == 1
```

Running the fraud detection tests:

```bash
$ pytest test_payment_gateway.py -k fraud -v
```

**Output**:

```text
test_payment_gateway.py::test_fraud_gateway_initialization PASSED        [ 25%]
test_payment_gateway.py::test_fraud_detection_high_amount PASSED         [ 50%]
test_payment_gateway.py::test_fraud_detection_normal_amount PASSED       [ 75%]
test_payment_gateway.py::test_fraud_detection_repeated_card PASSED       [100%]

========================== 4 passed in 0.02s ==========================
```

Excellent! Our tests verify that:
1. The subclass inherits base functionality correctly
2. The subclass adds new behavior (fraud detection)
3. The subclass overrides methods appropriately

### The Journey: From Problem to Solution

| Iteration | Problem                          | Solution Applied                | Result                    |
|-----------|----------------------------------|---------------------------------|---------------------------|
| 0         | No tests for class               | Write first test                | Basic coverage            |
| 1         | Repeated instantiation           | Create fixture                  | Reusable setup            |
| 2         | Unknown isolation behavior       | Test state isolation            | Verified isolation        |
| 3         | Need to test state accumulation  | Test multiple transactions      | State tracking verified   |
| 4         | Need to test method interactions | Test success/failure recording  | Integration verified      |
| 5         | Need individual method tests     | Test each method separately     | Complete coverage         |
| 6         | Need to test inheritance         | Create subclass and test it     | Inheritance verified      |

### Decision Framework: Testing Class Methods

**When to test methods individually**:
- The method has complex logic independent of other methods
- The method is public API that users will call directly
- You want to test edge cases that are hard to trigger through high-level methods
- The method's behavior is worth documenting separately

**When to test methods through integration**:
- Methods are simple and unlikely to fail independently
- The interaction between methods is the primary concern
- Testing individually would duplicate coverage
- Methods are private implementation details

**For our payment gateway**:
- We test `validate_credit_card()` and `calculate_fee()` individually because they have independent logic
- We test `process_payment()` both individually and through state-tracking methods
- We test state-tracking methods (`get_transaction_count()`, `get_total_processed()`) through integration

### Lessons Learned

1. **Use fixtures for class instantiation**: Avoid repeating setup code
2. **Test state isolation**: Verify that tests don't interfere with each other
3. **Test state accumulation**: Verify that objects maintain state correctly
4. **Test method interactions**: Verify that methods work together correctly
5. **Test inheritance carefully**: Verify both inherited and overridden behavior
6. **Test initialization**: Verify that objects start in the correct state
7. **Use descriptive test names**: Make it clear what each test verifies

## Testing Private Methods (And Why You Might Not Want To)

## Testing Private Methods (And Why You Might Not Want To)

Python doesn't have true private methods—the single underscore (`_method`) and double underscore (`__method`) are conventions, not enforcement. This raises a question: should we test methods marked as "private"?

The short answer: **usually not directly**. But let's explore why, and when you might make exceptions.

### The Reference Implementation: A Gateway with Private Methods

Let's refactor our payment gateway to use private methods for internal logic:

```python
# payment_gateway.py

class PaymentGateway:
    """A payment gateway that processes credit card transactions."""
    
    def __init__(self, merchant_id: str, api_key: str):
        self.merchant_id = merchant_id
        self.api_key = api_key
        self.transaction_history = []
        self.total_processed = 0.0
    
    def _validate_luhn(self, card_number: str) -> bool:
        """Private: Validate using Luhn algorithm."""
        digits = [int(d) for d in card_number]
        checksum = 0
        
        for i in range(len(digits) - 2, -1, -2):
            doubled = digits[i] * 2
            checksum += doubled if doubled < 10 else doubled - 9
        
        for i in range(len(digits) - 1, -1, -2):
            checksum += digits[i]
        
        return checksum % 10 == 0
    
    def _validate_format(self, card_number: str) -> bool:
        """Private: Validate card number format."""
        if not card_number.isdigit():
            return False
        
        if len(card_number) < 13 or len(card_number) > 19:
            return False
        
        return True
    
    def validate_credit_card(self, card_number: str) -> bool:
        """Public: Validate credit card number."""
        return self._validate_format(card_number) and self._validate_luhn(card_number)
    
    def _calculate_base_fee(self, amount: float) -> float:
        """Private: Calculate base processing fee."""
        return amount * 0.029
    
    def _calculate_card_type_fee(self, amount: float, card_type: str) -> float:
        """Private: Calculate card-type-specific fee."""
        if card_type == "amex":
            return amount * 0.005
        return 0.0
    
    def calculate_fee(self, amount: float, card_type: str) -> float:
        """Public: Calculate total processing fee."""
        base_fee = self._calculate_base_fee(amount)
        card_fee = self._calculate_card_type_fee(amount, card_type)
        transaction_fee = 0.30
        return base_fee + card_fee + transaction_fee
    
    def _record_transaction(self, card_number: str, amount: float, fee: float):
        """Private: Record a successful transaction."""
        transaction = {
            "card_last_four": card_number[-4:],
            "amount": amount,
            "fee": fee,
            "total": amount + fee
        }
        self.transaction_history.append(transaction)
        self.total_processed += amount + fee
    
    def process_payment(self, card_number: str, amount: float, card_type: str) -> dict:
        """Public: Process a payment transaction."""
        if not self.validate_credit_card(card_number):
            return {
                "success": False,
                "error": "Invalid credit card number"
            }
        
        if amount <= 0:
            return {
                "success": False,
                "error": "Amount must be positive"
            }
        
        fee = self.calculate_fee(amount, card_type)
        total = amount + fee
        
        self._record_transaction(card_number, amount, fee)
        
        return {
            "success": True,
            "amount": amount,
            "fee": fee,
            "total": total,
            "card_last_four": card_number[-4:]
        }
    
    def get_transaction_count(self) -> int:
        """Public: Get the number of processed transactions."""
        return len(self.transaction_history)
    
    def get_total_processed(self) -> float:
        """Public: Get the total amount processed."""
        return self.total_processed
```

Now we have several private methods:
- `_validate_luhn()` - Luhn algorithm implementation
- `_validate_format()` - Format validation
- `_calculate_base_fee()` - Base fee calculation
- `_calculate_card_type_fee()` - Card-type-specific fee
- `_record_transaction()` - Transaction recording

### Iteration 0: Testing Only Public Methods

Let's write tests that only use the public API:

```python
# test_payment_gateway.py

import pytest
from pytest import approx
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

def test_validate_credit_card_valid(gateway):
    """Test validation with valid card."""
    assert gateway.validate_credit_card("4532015112830366") == True

def test_validate_credit_card_invalid_luhn(gateway):
    """Test validation with invalid Luhn checksum."""
    assert gateway.validate_credit_card("1234567890123456") == False

def test_validate_credit_card_invalid_format(gateway):
    """Test validation with invalid format."""
    assert gateway.validate_credit_card("1234-5678-9012-3456") == False

def test_calculate_fee_visa(gateway):
    """Test fee calculation for Visa."""
    fee = gateway.calculate_fee(100.00, "visa")
    assert fee == approx(3.20)

def test_calculate_fee_amex(gateway):
    """Test fee calculation for Amex."""
    fee = gateway.calculate_fee(100.00, "amex")
    assert fee == approx(3.70)

def test_process_payment_records_transaction(gateway):
    """Test that successful payment is recorded."""
    gateway.process_payment("4532015112830366", 100.00, "visa")
    
    assert gateway.get_transaction_count() == 1
    assert gateway.get_total_processed() == approx(103.20)
```

Running these tests:

```bash
$ pytest test_payment_gateway.py -v
```

**Output**:

```text
test_payment_gateway.py::test_validate_credit_card_valid PASSED          [ 16%]
test_payment_gateway.py::test_validate_credit_card_invalid_luhn PASSED   [ 33%]
test_payment_gateway.py::test_validate_credit_card_invalid_format PASSED [ 50%]
test_payment_gateway.py::test_calculate_fee_visa PASSED                  [ 66%]
test_payment_gateway.py::test_calculate_fee_amex PASSED                  [ 83%]
test_payment_gateway.py::test_process_payment_records_transaction PASSED [100%]

========================== 6 passed in 0.02s ==========================
```

Perfect! All our tests pass, and we've achieved good coverage without testing private methods directly.

**Key insight**: By testing the public API, we've indirectly tested all the private methods. If `_validate_luhn()` is broken, `validate_credit_card()` will fail. If `_record_transaction()` is broken, our state-tracking tests will fail.

### When Testing Private Methods Makes Sense

But what if a private method has complex logic that's hard to test through the public API? Let's add a more complex private method:

```python
# payment_gateway.py

class PaymentGateway:
    # ... (previous code remains the same)
    
    def _detect_card_type(self, card_number: str) -> str:
        """Private: Detect card type from card number."""
        # Visa: starts with 4
        if card_number[0] == "4":
            return "visa"
        
        # Mastercard: starts with 51-55 or 2221-2720
        if card_number[:2] in ["51", "52", "53", "54", "55"]:
            return "mastercard"
        if 2221 <= int(card_number[:4]) <= 2720:
            return "mastercard"
        
        # Amex: starts with 34 or 37
        if card_number[:2] in ["34", "37"]:
            return "amex"
        
        # Discover: starts with 6011, 622126-622925, 644-649, or 65
        if card_number[:4] == "6011":
            return "discover"
        if 622126 <= int(card_number[:6]) <= 622925:
            return "discover"
        if card_number[:3] in ["644", "645", "646", "647", "648", "649"]:
            return "discover"
        if card_number[:2] == "65":
            return "discover"
        
        return "unknown"
    
    def process_payment(self, card_number: str, amount: float, card_type: str = None) -> dict:
        """Public: Process a payment transaction."""
        if not self.validate_credit_card(card_number):
            return {
                "success": False,
                "error": "Invalid credit card number"
            }
        
        if amount <= 0:
            return {
                "success": False,
                "error": "Amount must be positive"
            }
        
        # Auto-detect card type if not provided
        if card_type is None:
            card_type = self._detect_card_type(card_number)
        
        fee = self.calculate_fee(amount, card_type)
        total = amount + fee
        
        self._record_transaction(card_number, amount, fee)
        
        return {
            "success": True,
            "amount": amount,
            "fee": fee,
            "total": total,
            "card_last_four": card_number[-4:],
            "card_type": card_type
        }
```

Now we have a complex private method `_detect_card_type()` with many edge cases. Testing all these cases through `process_payment()` would require many valid card numbers for each type.

### Iteration 1: Testing Private Methods Directly

Let's test the private method directly:

```python
# test_payment_gateway.py

import pytest
from pytest import approx
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

# Tests for private method _detect_card_type()
def test_detect_card_type_visa(gateway):
    """Test Visa detection."""
    assert gateway._detect_card_type("4532015112830366") == "visa"

def test_detect_card_type_mastercard_51(gateway):
    """Test Mastercard detection (51-55 range)."""
    assert gateway._detect_card_type("5425233430109903") == "mastercard"

def test_detect_card_type_mastercard_2221(gateway):
    """Test Mastercard detection (2221-2720 range)."""
    assert gateway._detect_card_type("2221000000000009") == "mastercard"

def test_detect_card_type_amex(gateway):
    """Test Amex detection."""
    assert gateway._detect_card_type("378282246310005") == "amex"

def test_detect_card_type_discover_6011(gateway):
    """Test Discover detection (6011 prefix)."""
    assert gateway._detect_card_type("6011111111111117") == "discover"

def test_detect_card_type_discover_65(gateway):
    """Test Discover detection (65 prefix)."""
    assert gateway._detect_card_type("6500000000000002") == "discover"

def test_detect_card_type_unknown(gateway):
    """Test unknown card type."""
    assert gateway._detect_card_type("9999999999999999") == "unknown"
```

Running these tests:

```bash
$ pytest test_payment_gateway.py -k detect -v
```

**Output**:

```text
test_payment_gateway.py::test_detect_card_type_visa PASSED               [ 14%]
test_payment_gateway.py::test_detect_card_type_mastercard_51 PASSED      [ 28%]
test_payment_gateway.py::test_detect_card_type_mastercard_2221 PASSED    [ 42%]
test_payment_gateway.py::test_detect_card_type_amex PASSED               [ 57%]
test_payment_gateway.py::test_detect_card_type_discover_6011 PASSED      [ 71%]
test_payment_gateway.py::test_detect_card_type_discover_65 PASSED        [ 85%]
test_payment_gateway.py::test_detect_card_type_unknown PASSED            [100%]

========================== 7 passed in 0.02s ==========================
```

Good! We've tested the complex private method directly. But now we have a problem: we're testing implementation details. If we refactor `_detect_card_type()` to use a different algorithm, our tests will break even if the public behavior is unchanged.

### The Trade-off: Implementation Testing vs. Behavior Testing

**Testing private methods directly**:

**Pros**:
- Easier to test complex logic in isolation
- More granular failure messages
- Can test edge cases that are hard to trigger through public API

**Cons**:
- Tests become coupled to implementation
- Refactoring becomes harder
- Tests may pass even if public behavior is broken

**Testing only public methods**:

**Pros**:
- Tests verify actual user-facing behavior
- Refactoring is easier (tests don't break)
- Tests document the public API

**Cons**:
- Complex private logic may be under-tested
- Failure messages may be less specific
- May require more setup to trigger edge cases

### Iteration 2: A Hybrid Approach

Let's use a hybrid approach: test the public API thoroughly, but add a few targeted tests for complex private methods:

```python
# test_payment_gateway.py

import pytest
from pytest import approx
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

# Primary tests: Public API
def test_process_payment_auto_detect_visa(gateway):
    """Test payment processing with auto-detected Visa."""
    result = gateway.process_payment("4532015112830366", 100.00)
    
    assert result["success"] == True
    assert result["card_type"] == "visa"
    assert result["fee"] == approx(3.20)

def test_process_payment_auto_detect_amex(gateway):
    """Test payment processing with auto-detected Amex."""
    result = gateway.process_payment("378282246310005", 100.00)
    
    assert result["success"] == True
    assert result["card_type"] == "amex"
    assert result["fee"] == approx(3.70)

def test_process_payment_explicit_card_type(gateway):
    """Test payment processing with explicit card type."""
    result = gateway.process_payment("4532015112830366", 100.00, card_type="visa")
    
    assert result["success"] == True
    assert result["card_type"] == "visa"

# Supplementary tests: Complex private method
def test_detect_card_type_edge_cases(gateway):
    """Test card type detection edge cases."""
    # These are hard to test through public API because they require
    # valid card numbers for each edge case
    
    # Mastercard 2221-2720 range (new BIN range)
    assert gateway._detect_card_type("2221000000000009") == "mastercard"
    
    # Discover 622126-622925 range
    assert gateway._detect_card_type("6221260000000000") == "discover"
    
    # Unknown card type
    assert gateway._detect_card_type("9999999999999999") == "unknown"
```

Running all tests:

```bash
$ pytest test_payment_gateway.py -v
```

**Output**:

```text
test_payment_gateway.py::test_process_payment_auto_detect_visa PASSED    [ 33%]
test_payment_gateway.py::test_process_payment_auto_detect_amex PASSED    [ 66%]
test_payment_gateway.py::test_process_payment_explicit_card_type PASSED  [100%]
test_payment_gateway.py::test_detect_card_type_edge_cases PASSED         [100%]

========================== 4 passed in 0.02s ==========================
```

This hybrid approach gives us:
1. **Primary coverage** through public API tests
2. **Supplementary coverage** for complex edge cases in private methods
3. **Flexibility** to refactor without breaking most tests

### When to Test Private Methods

**Test private methods directly when**:
- The method has complex logic with many edge cases
- Testing through public API would require excessive setup
- The method is critical to correctness (e.g., security, financial calculations)
- The method is unlikely to change (stable implementation)

**Don't test private methods when**:
- The logic is simple and well-covered by public API tests
- The method is likely to be refactored
- Testing through public API is straightforward
- The method is just a helper for code organization

### Alternative: Make It Public

If you find yourself wanting to test a private method extensively, consider whether it should be public:

```python
# payment_gateway.py

class CardTypeDetector:
    """Utility class for detecting card types."""
    
    @staticmethod
    def detect(card_number: str) -> str:
        """Detect card type from card number."""
        # Visa: starts with 4
        if card_number[0] == "4":
            return "visa"
        
        # Mastercard: starts with 51-55 or 2221-2720
        if card_number[:2] in ["51", "52", "53", "54", "55"]:
            return "mastercard"
        if 2221 <= int(card_number[:4]) <= 2720:
            return "mastercard"
        
        # Amex: starts with 34 or 37
        if card_number[:2] in ["34", "37"]:
            return "amex"
        
        # Discover: starts with 6011, 622126-622925, 644-649, or 65
        if card_number[:4] == "6011":
            return "discover"
        if 622126 <= int(card_number[:6]) <= 622925:
            return "discover"
        if card_number[:3] in ["644", "645", "646", "647", "648", "649"]:
            return "discover"
        if card_number[:2] == "65":
            return "discover"
        
        return "unknown"

class PaymentGateway:
    """A payment gateway that processes credit card transactions."""
    
    def __init__(self, merchant_id: str, api_key: str):
        self.merchant_id = merchant_id
        self.api_key = api_key
        self.transaction_history = []
        self.total_processed = 0.0
        self.card_detector = CardTypeDetector()
    
    # ... (other methods remain the same)
    
    def process_payment(self, card_number: str, amount: float, card_type: str = None) -> dict:
        """Public: Process a payment transaction."""
        if not self.validate_credit_card(card_number):
            return {
                "success": False,
                "error": "Invalid credit card number"
            }
        
        if amount <= 0:
            return {
                "success": False,
                "error": "Amount must be positive"
            }
        
        # Auto-detect card type if not provided
        if card_type is None:
            card_type = self.card_detector.detect(card_number)
        
        fee = self.calculate_fee(amount, card_type)
        total = amount + fee
        
        self._record_transaction(card_number, amount, fee)
        
        return {
            "success": True,
            "amount": amount,
            "fee": fee,
            "total": total,
            "card_last_four": card_number[-4:],
            "card_type": card_type
        }
```

Now we can test `CardTypeDetector` as a public utility class:

```python
# test_card_detector.py

from payment_gateway import CardTypeDetector

def test_detect_visa():
    assert CardTypeDetector.detect("4532015112830366") == "visa"

def test_detect_mastercard_51():
    assert CardTypeDetector.detect("5425233430109903") == "mastercard"

def test_detect_mastercard_2221():
    assert CardTypeDetector.detect("2221000000000009") == "mastercard"

def test_detect_amex():
    assert CardTypeDetector.detect("378282246310005") == "amex"

def test_detect_discover():
    assert CardTypeDetector.detect("6011111111111117") == "discover"

def test_detect_unknown():
    assert CardTypeDetector.detect("9999999999999999") == "unknown"
```

This approach:
- Makes the complex logic testable without breaking encapsulation
- Allows the logic to be reused elsewhere
- Makes the code more modular and maintainable

### Lessons Learned

1. **Prefer testing public API**: Test behavior, not implementation
2. **Private methods are tested indirectly**: If public tests pass, private methods work
3. **Test complex private methods sparingly**: Only when necessary for edge case coverage
4. **Consider making it public**: If you need extensive testing, maybe it should be public
5. **Use hybrid approach**: Primary coverage through public API, supplementary for edge cases
6. **Extract to utility classes**: Complex logic often deserves its own testable class
7. **Balance pragmatism and purity**: Sometimes testing private methods is the practical choice

## Testing Code with Side Effects

## Testing Code with Side Effects

So far, we've tested pure functions and methods that return values. But real-world code has side effects: it writes to databases, sends emails, logs messages, modifies files, and makes network requests. How do we test code that changes the world outside our program?

### The Reference Implementation: A Payment Gateway with Side Effects

Let's extend our payment gateway to include realistic side effects:

```python
# payment_gateway.py

import logging
from datetime import datetime

class PaymentGateway:
    """A payment gateway that processes credit card transactions."""
    
    def __init__(self, merchant_id: str, api_key: str, logger=None):
        self.merchant_id = merchant_id
        self.api_key = api_key
        self.transaction_history = []
        self.total_processed = 0.0
        self.logger = logger or logging.getLogger(__name__)
    
    def validate_credit_card(self, card_number: str) -> bool:
        """Validate credit card number using Luhn algorithm."""
        if not card_number.isdigit():
            self.logger.warning(f"Invalid card format: non-numeric characters")
            return False
        
        if len(card_number) < 13 or len(card_number) > 19:
            self.logger.warning(f"Invalid card format: length {len(card_number)}")
            return False
        
        # Luhn algorithm
        digits = [int(d) for d in card_number]
        checksum = 0
        
        for i in range(len(digits) - 2, -1, -2):
            doubled = digits[i] * 2
            checksum += doubled if doubled < 10 else doubled - 9
        
        for i in range(len(digits) - 1, -1, -2):
            checksum += digits[i]
        
        is_valid = checksum % 10 == 0
        if not is_valid:
            self.logger.warning(f"Invalid card: failed Luhn check")
        
        return is_valid
    
    def calculate_fee(self, amount: float, card_type: str) -> float:
        """Calculate processing fee based on amount and card type."""
        base_fee = amount * 0.029
        
        if card_type == "amex":
            base_fee += amount * 0.005
        
        return base_fee + 0.30
    
    def _send_receipt_email(self, email: str, transaction: dict):
        """Send receipt email to customer."""
        # In real code, this would send an actual email
        self.logger.info(f"Sending receipt to {email} for transaction {transaction['card_last_four']}")
    
    def _notify_fraud_team(self, transaction: dict, reason: str):
        """Notify fraud team of suspicious transaction."""
        # In real code, this would send a notification
        self.logger.warning(f"FRAUD ALERT: {reason} - Card {transaction['card_last_four']}")
    
    def process_payment(self, card_number: str, amount: float, card_type: str, 
                       customer_email: str = None) -> dict:
        """Process a payment transaction."""
        self.logger.info(f"Processing payment: ${amount:.2f} on {card_type}")
        
        if not self.validate_credit_card(card_number):
            self.logger.error(f"Payment failed: invalid card")
            return {
                "success": False,
                "error": "Invalid credit card number"
            }
        
        if amount <= 0:
            self.logger.error(f"Payment failed: invalid amount ${amount:.2f}")
            return {
                "success": False,
                "error": "Amount must be positive"
            }
        
        # Check for suspicious activity
        if amount > 10000:
            transaction = {
                "card_last_four": card_number[-4:],
                "amount": amount
            }
            self._notify_fraud_team(transaction, "High-value transaction")
        
        fee = self.calculate_fee(amount, card_type)
        total = amount + fee
        
        # Record transaction
        transaction = {
            "card_last_four": card_number[-4:],
            "amount": amount,
            "fee": fee,
            "total": total,
            "timestamp": datetime.now().isoformat()
        }
        self.transaction_history.append(transaction)
        self.total_processed += total
        
        self.logger.info(f"Payment successful: ${total:.2f} total")
        
        # Send receipt if email provided
        if customer_email:
            self._send_receipt_email(customer_email, transaction)
        
        return {
            "success": True,
            "amount": amount,
            "fee": fee,
            "total": total,
            "card_last_four": card_number[-4:]
        }
    
    def get_transaction_count(self) -> int:
        """Get the number of processed transactions."""
        return len(self.transaction_history)
    
    def get_total_processed(self) -> float:
        """Get the total amount processed."""
        return self.total_processed
```

Now our gateway has several side effects:
- **Logging**: Records events at various log levels
- **Email sending**: Sends receipts to customers
- **Fraud notifications**: Alerts the fraud team
- **Timestamps**: Records when transactions occur

### Iteration 0: Testing Without Considering Side Effects

Let's write a basic test:

```python
# test_payment_gateway.py

import pytest
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

def test_process_payment_success(gateway):
    result = gateway.process_payment("4532015112830366", 100.00, "visa")
    assert result["success"] == True
```

Running this test:

```bash
$ pytest test_payment_gateway.py::test_process_payment_success -v -s
```

**Output**:

```text
test_payment_gateway.py::test_process_payment_success INFO:payment_gateway:Processing payment: $100.00 on visa
INFO:payment_gateway:Payment successful: $103.20 total
PASSED

========================== 1 passed in 0.01s ===========================
```

The test passes, but notice the log messages appearing in the output. This is a side effect we're not controlling or verifying.

**Current limitation**: We're not testing that logging happens correctly, and log messages clutter our test output.

### Iteration 1: Capturing and Verifying Log Messages

Let's use pytest's `caplog` fixture to capture and verify log messages:

```python
# test_payment_gateway.py

import pytest
import logging
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

def test_process_payment_logs_success(gateway, caplog):
    """Test that successful payment is logged."""
    with caplog.at_level(logging.INFO):
        result = gateway.process_payment("4532015112830366", 100.00, "visa")
    
    assert result["success"] == True
    
    # Verify log messages
    assert "Processing payment: $100.00 on visa" in caplog.text
    assert "Payment successful: $103.20 total" in caplog.text

def test_process_payment_logs_invalid_card(gateway, caplog):
    """Test that invalid card is logged."""
    with caplog.at_level(logging.WARNING):
        result = gateway.process_payment("1234567890123456", 100.00, "visa")
    
    assert result["success"] == False
    
    # Verify warning was logged
    assert "Invalid card: failed Luhn check" in caplog.text
    assert "Payment failed: invalid card" in caplog.text
```

Running these tests:

```bash
$ pytest test_payment_gateway.py -v
```

**Output**:

```text
test_payment_gateway.py::test_process_payment_logs_success PASSED        [ 50%]
test_payment_gateway.py::test_process_payment_logs_invalid_card PASSED   [100%]

========================== 2 passed in 0.02s ==========================
```

Excellent! We're now verifying that logging happens correctly. The `caplog` fixture captures all log messages, and we can assert on their content.

**Current state**: We can test logging side effects. But what about other side effects like email sending?

### Iteration 2: Testing Email Side Effects with Mocking

We can't send real emails in tests, so we need to mock the email sending:

```python
# test_payment_gateway.py

import pytest
import logging
from unittest.mock import Mock, patch
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

def test_process_payment_sends_receipt(gateway):
    """Test that receipt email is sent when email provided."""
    # Mock the email sending method
    gateway._send_receipt_email = Mock()
    
    result = gateway.process_payment(
        "4532015112830366", 
        100.00, 
        "visa",
        customer_email="customer@example.com"
    )
    
    assert result["success"] == True
    
    # Verify email was sent
    gateway._send_receipt_email.assert_called_once()
    
    # Verify email was sent with correct arguments
    call_args = gateway._send_receipt_email.call_args
    assert call_args[0][0] == "customer@example.com"  # First positional arg
    assert call_args[0][1]["card_last_four"] == "0366"  # Second positional arg

def test_process_payment_no_receipt_without_email(gateway):
    """Test that no receipt is sent when email not provided."""
    gateway._send_receipt_email = Mock()
    
    result = gateway.process_payment("4532015112830366", 100.00, "visa")
    
    assert result["success"] == True
    
    # Verify email was NOT sent
    gateway._send_receipt_email.assert_not_called()
```

Running these tests:

```bash
$ pytest test_payment_gateway.py -k receipt -v
```

**Output**:

```text
test_payment_gateway.py::test_process_payment_sends_receipt PASSED       [ 50%]
test_payment_gateway.py::test_process_payment_no_receipt_without_email PASSED [100%]

========================== 2 passed in 0.02s ==========================
```

Perfect! We're now testing that email sending happens (or doesn't happen) correctly without actually sending emails.

**Current state**: We can test both logging and email side effects. But what about fraud notifications?

### Iteration 3: Testing Conditional Side Effects

Let's test that fraud notifications are sent for high-value transactions:

```python
# test_payment_gateway.py

import pytest
import logging
from unittest.mock import Mock
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

def test_high_value_transaction_triggers_fraud_alert(gateway):
    """Test that high-value transactions trigger fraud alerts."""
    gateway._notify_fraud_team = Mock()
    
    result = gateway.process_payment("4532015112830366", 15000.00, "visa")
    
    assert result["success"] == True
    
    # Verify fraud team was notified
    gateway._notify_fraud_team.assert_called_once()
    
    # Verify notification details
    call_args = gateway._notify_fraud_team.call_args
    transaction = call_args[0][0]
    reason = call_args[0][1]
    
    assert transaction["card_last_four"] == "0366"
    assert transaction["amount"] == 15000.00
    assert reason == "High-value transaction"

def test_normal_transaction_no_fraud_alert(gateway):
    """Test that normal transactions don't trigger fraud alerts."""
    gateway._notify_fraud_team = Mock()
    
    result = gateway.process_payment("4532015112830366", 100.00, "visa")
    
    assert result["success"] == True
    
    # Verify fraud team was NOT notified
    gateway._notify_fraud_team.assert_not_called()
```

Running these tests:

```bash
$ pytest test_payment_gateway.py -k fraud -v
```

**Output**:

```text
test_payment_gateway.py::test_high_value_transaction_triggers_fraud_alert PASSED [ 50%]
test_payment_gateway.py::test_normal_transaction_no_fraud_alert PASSED   [100%]

========================== 2 passed in 0.02s ==========================
```

Excellent! We're now testing conditional side effects—side effects that only happen under certain conditions.

### Iteration 4: Testing Time-Dependent Side Effects

Our gateway records timestamps for transactions. How do we test this without making our tests dependent on the current time?

```python
# test_payment_gateway.py

import pytest
from unittest.mock import Mock, patch
from datetime import datetime
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

def test_transaction_includes_timestamp(gateway):
    """Test that transactions include timestamps."""
    # Mock datetime to return a fixed time
    fixed_time = datetime(2024, 1, 15, 10, 30, 0)
    
    with patch('payment_gateway.datetime') as mock_datetime:
        mock_datetime.now.return_value = fixed_time
        
        result = gateway.process_payment("4532015112830366", 100.00, "visa")
    
    assert result["success"] == True
    
    # Verify transaction has timestamp
    transaction = gateway.transaction_history[0]
    assert transaction["timestamp"] == "2024-01-15T10:30:00"

def test_multiple_transactions_have_different_timestamps(gateway):
    """Test that multiple transactions have different timestamps."""
    # Mock datetime to return incrementing times
    times = [
        datetime(2024, 1, 15, 10, 30, 0),
        datetime(2024, 1, 15, 10, 31, 0),
        datetime(2024, 1, 15, 10, 32, 0)
    ]
    
    with patch('payment_gateway.datetime') as mock_datetime:
        mock_datetime.now.side_effect = times
        
        gateway.process_payment("4532015112830366", 100.00, "visa")
        gateway.process_payment("4532015112830366", 50.00, "visa")
        gateway.process_payment("4532015112830366", 25.00, "visa")
    
    # Verify each transaction has correct timestamp
    assert gateway.transaction_history[0]["timestamp"] == "2024-01-15T10:30:00"
    assert gateway.transaction_history[1]["timestamp"] == "2024-01-15T10:31:00"
    assert gateway.transaction_history[2]["timestamp"] == "2024-01-15T10:32:00"
```

Running these tests:

```bash
$ pytest test_payment_gateway.py -k timestamp -v
```

**Output**:

```text
test_payment_gateway.py::test_transaction_includes_timestamp PASSED      [ 50%]
test_payment_gateway.py::test_multiple_transactions_have_different_timestamps PASSED [100%]

========================== 2 passed in 0.02s ==========================
```

Perfect! We're now testing time-dependent side effects by mocking the datetime module.

### Common Failure Modes and Their Signatures

#### Symptom: Test passes but side effect doesn't happen in production

**Pytest output pattern**:
```
test_payment_gateway.py::test_email_sent PASSED
```

**Diagnostic clues**:
- Test passes but users report not receiving emails
- Mock was called but real implementation wasn't tested

**Root cause**: Over-mocking—we mocked the side effect but never verified the real implementation works

**Solution**: Add integration tests that verify real side effects in a test environment

#### Symptom: Tests fail intermittently due to timing

**Pytest output pattern**:
```
test_payment_gateway.py::test_timestamp FAILED
AssertionError: assert '2024-01-15T10:30:00.123456' == '2024-01-15T10:30:00.123457'
```

**Diagnostic clues**:
- Timestamp assertions fail randomly
- Tests pass when run individually but fail in suite

**Root cause**: Time-dependent code without mocking

**Solution**: Mock datetime or use approximate assertions for timestamps

#### Symptom: Log messages not captured

**Pytest output pattern**:
```
test_payment_gateway.py::test_logging FAILED
AssertionError: assert 'Payment successful' in ''
```

**Diagnostic clues**:
- `caplog.text` is empty
- Log messages appear in console but not in caplog

**Root cause**: Logger not configured correctly or wrong log level

**Solution**: Use `caplog.at_level()` with correct log level

### The Journey: From Problem to Solution

| Iteration | Problem                          | Solution Applied                | Result                    |
|-----------|----------------------------------|---------------------------------|---------------------------|
| 0         | Side effects not tested          | Basic test                      | Test passes but incomplete|
| 1         | Logging not verified             | Use caplog fixture              | Logging verified          |
| 2         | Email sending not tested         | Mock email method               | Email behavior verified   |
| 3         | Conditional side effects         | Mock and verify conditions      | Conditional logic tested  |
| 4         | Time-dependent side effects      | Mock datetime                   | Timestamps testable       |

### Final Implementation

Here's our complete test suite for side effects:

```python
# test_payment_gateway.py

import pytest
import logging
from unittest.mock import Mock, patch
from datetime import datetime
from payment_gateway import PaymentGateway

@pytest.fixture
def gateway():
    """Provide a PaymentGateway instance for testing."""
    return PaymentGateway("MERCHANT123", "secret_key")

# Logging tests
def test_process_payment_logs_success(gateway, caplog):
    """Test that successful payment is logged."""
    with caplog.at_level(logging.INFO):
        result = gateway.process_payment("4532015112830366", 100.00, "visa")
    
    assert result["success"] == True
    assert "Processing payment: $100.00 on visa" in caplog.text
    assert "Payment successful: $103.20 total" in caplog.text

def test_process_payment_logs_invalid_card(gateway, caplog):
    """Test that invalid card is logged."""
    with caplog.at_level(logging.WARNING):
        result = gateway.process_payment("1234567890123456", 100.00, "visa")
    
    assert result["success"] == False
    assert "Invalid card: failed Luhn check" in caplog.text

# Email tests
def test_process_payment_sends_receipt(gateway):
    """Test that receipt email is sent when email provided."""
    gateway._send_receipt_email = Mock()
    
    result = gateway.process_payment(
        "4532015112830366", 
        100.00, 
        "visa",
        customer_email="customer@example.com"
    )
    
    assert result["success"] == True
    gateway._send_receipt_email.assert_called_once()

def test_process_payment_no_receipt_without_email(gateway):
    """Test that no receipt is sent when email not provided."""
    gateway._send_receipt_email = Mock()
    
    result = gateway.process_payment("4532015112830366", 100.00, "visa")
    
    assert result["success"] == True
    gateway._send_receipt_email.assert_not_called()

# Fraud detection tests
def test_high_value_transaction_triggers_fraud_alert(gateway):
    """Test that high-value transactions trigger fraud alerts."""
    gateway._notify_fraud_team = Mock()
    
    result = gateway.process_payment("4532015112830366", 15000.00, "visa")
    
    assert result["success"] == True
    gateway._notify_fraud_team.assert_called_once()

def test_normal_transaction_no_fraud_alert(gateway):
    """Test that normal transactions don't trigger fraud alerts."""
    gateway._notify_fraud_team = Mock()
    
    result = gateway.process_payment("4532015112830366", 100.00, "visa")
    
    assert result["success"] == True
    gateway._notify_fraud_team.assert_not_called()

# Timestamp tests
def test_transaction_includes_timestamp(gateway):
    """Test that transactions include timestamps."""
    fixed_time = datetime(2024, 1, 15, 10, 30, 0)
    
    with patch('payment_gateway.datetime') as mock_datetime:
        mock_datetime.now.return_value = fixed_time
        
        result = gateway.process_payment("4532015112830366", 100.00, "visa")
    
    assert result["success"] == True
    transaction = gateway.transaction_history[0]
    assert transaction["timestamp"] == "2024-01-15T10:30:00"
```

### Decision Framework: Testing Side Effects

**When to use caplog**:
- Testing that logging happens at correct levels
- Verifying log message content
- Testing error logging and warnings

**When to use mocks**:
- Testing external service calls (email, SMS, API)
- Testing file system operations
- Testing database writes
- Testing any side effect that shouldn't happen in tests

**When to use patches**:
- Testing time-dependent code (datetime, time.sleep)
- Testing random behavior (random.random, uuid.uuid4)
- Testing environment-dependent code (os.environ)

**When to use real implementations**:
- Integration tests in test environment
- Testing critical paths end-to-end
- Verifying that mocked behavior matches reality

### Lessons Learned

1. **Use caplog for logging**: Pytest's caplog fixture makes testing logs easy
2. **Mock external side effects**: Don't send real emails or make real API calls in tests
3. **Verify mock calls**: Use `assert_called_once()`, `assert_called_with()`, etc.
4. **Mock time-dependent code**: Use `patch` to control datetime and other time sources
5. **Test conditional side effects**: Verify side effects happen (or don't) based on conditions
6. **Balance mocking and integration**: Mock for unit tests, use real implementations for integration tests
7. **Document what you're mocking**: Make it clear why each mock exists

## Integration Testing Within Your Codebase

## Integration Testing Within Your Codebase

We've tested individual functions, classes, and side effects. But real applications are systems of components working together. Integration testing verifies that these components interact correctly.

Unlike unit tests that isolate components, integration tests verify the **seams** between components—the places where things can go wrong even when individual pieces work correctly.

### The Reference Implementation: A Complete Payment System

Let's build a complete payment system with multiple components:

```python
# payment_system.py

import logging
from datetime import datetime
from typing import Optional

class CardValidator:
    """Validates credit card numbers."""
    
    def __init__(self, logger=None):
        self.logger = logger or logging.getLogger(__name__)
    
    def validate(self, card_number: str) -> tuple[bool, Optional[str]]:
        """Validate card number. Returns (is_valid, error_message)."""
        if not card_number.isdigit():
            self.logger.warning("Card validation failed: non-numeric")
            return False, "Card number must contain only digits"
        
        if len(card_number) < 13 or len(card_number) > 19:
            self.logger.warning(f"Card validation failed: invalid length {len(card_number)}")
            return False, "Card number must be 13-19 digits"
        
        # Luhn algorithm
        digits = [int(d) for d in card_number]
        checksum = 0
        
        for i in range(len(digits) - 2, -1, -2):
            doubled = digits[i] * 2
            checksum += doubled if doubled < 10 else doubled - 9
        
        for i in range(len(digits) - 1, -1, -2):
            checksum += digits[i]
        
        if checksum % 10 != 0:
            self.logger.warning("Card validation failed: Luhn check")
            return False, "Invalid card number"
        
        return True, None

class FeeCalculator:
    """Calculates processing fees."""
    
    def __init__(self, base_rate: float = 0.029, transaction_fee: float = 0.30):
        self.base_rate = base_rate
        self.transaction_fee = transaction_fee
        self.card_type_rates = {
            "amex": 0.005,
            "visa": 0.0,
            "mastercard": 0.0,
            "discover": 0.0
        }
    
    def calculate(self, amount: float, card_type: str) -> float:
        """Calculate total fee for transaction."""
        base_fee = amount * self.base_rate
        card_fee = amount * self.card_type_rates.get(card_type, 0.0)
        return base_fee + card_fee + self.transaction_fee

class TransactionRepository:
    """Stores transaction records."""
    
    def __init__(self):
        self.transactions = []
    
    def save(self, transaction: dict) -> str:
        """Save transaction and return transaction ID."""
        transaction_id = f"TXN{len(self.transactions) + 1:06d}"
        transaction["id"] = transaction_id
        transaction["timestamp"] = datetime.now().isoformat()
        self.transactions.append(transaction)
        return transaction_id
    
    def get_by_id(self, transaction_id: str) -> Optional[dict]:
        """Retrieve transaction by ID."""
        for txn in self.transactions:
            if txn["id"] == transaction_id:
                return txn
        return None
    
    def get_all(self) -> list[dict]:
        """Retrieve all transactions."""
        return self.transactions.copy()
    
    def get_total_processed(self) -> float:
        """Get total amount processed."""
        return sum(txn["total"] for txn in self.transactions)

class NotificationService:
    """Sends notifications about transactions."""
    
    def __init__(self, logger=None):
        self.logger = logger or logging.getLogger(__name__)
        self.sent_notifications = []
    
    def send_receipt(self, email: str, transaction: dict):
        """Send receipt email."""
        self.logger.info(f"Sending receipt to {email} for transaction {transaction['id']}")
        self.sent_notifications.append({
            "type": "receipt",
            "email": email,
            "transaction_id": transaction["id"]
        })
    
    def send_fraud_alert(self, transaction: dict, reason: str):
        """Send fraud alert."""
        self.logger.warning(f"FRAUD ALERT: {reason} - Transaction {transaction['id']}")
        self.sent_notifications.append({
            "type": "fraud_alert",
            "transaction_id": transaction["id"],
            "reason": reason
        })

class PaymentProcessor:
    """Orchestrates payment processing using multiple components."""
    
    def __init__(
        self,
        validator: CardValidator,
        fee_calculator: FeeCalculator,
        repository: TransactionRepository,
        notification_service: NotificationService,
        fraud_threshold: float = 10000.0,
        logger=None
    ):
        self.validator = validator
        self.fee_calculator = fee_calculator
        self.repository = repository
        self.notification_service = notification_service
        self.fraud_threshold = fraud_threshold
        self.logger = logger or logging.getLogger(__name__)
    
    def process_payment(
        self,
        card_number: str,
        amount: float,
        card_type: str,
        customer_email: Optional[str] = None
    ) -> dict:
        """Process a payment transaction."""
        self.logger.info(f"Processing payment: ${amount:.2f} on {card_type}")
        
        # Validate card
        is_valid, error = self.validator.validate(card_number)
        if not is_valid:
            self.logger.error(f"Payment failed: {error}")
            return {
                "success": False,
                "error": error
            }
        
        # Validate amount
        if amount <= 0:
            self.logger.error(f"Payment failed: invalid amount ${amount:.2f}")
            return {
                "success": False,
                "error": "Amount must be positive"
            }
        
        # Calculate fee
        fee = self.fee_calculator.calculate(amount, card_type)
        total = amount + fee
        
        # Create transaction record
        transaction = {
            "card_last_four": card_number[-4:],
            "amount": amount,
            "fee": fee,
            "total": total,
            "card_type": card_type
        }
        
        # Save transaction
        transaction_id = self.repository.save(transaction)
        transaction["id"] = transaction_id
        
        self.logger.info(f"Payment successful: {transaction_id} - ${total:.2f}")
        
        # Check for fraud
        if amount > self.fraud_threshold:
            self.notification_service.send_fraud_alert(
                transaction,
                f"High-value transaction: ${amount:.2f}"
            )
        
        # Send receipt
        if customer_email:
            self.notification_service.send_receipt(customer_email, transaction)
        
        return {
            "success": True,
            "transaction_id": transaction_id,
            "amount": amount,
            "fee": fee,
            "total": total,
            "card_last_four": card_number[-4:]
        }
    
    def get_transaction(self, transaction_id: str) -> Optional[dict]:
        """Retrieve a transaction by ID."""
        return self.repository.get_by_id(transaction_id)
    
    def get_total_processed(self) -> float:
        """Get total amount processed."""
        return self.repository.get_total_processed()
```

This is a realistic payment system with five components:
1. **CardValidator**: Validates credit cards
2. **FeeCalculator**: Calculates fees
3. **TransactionRepository**: Stores transactions
4. **NotificationService**: Sends notifications
5. **PaymentProcessor**: Orchestrates everything

### Iteration 0: Testing Components in Isolation

First, let's verify each component works independently:

```python
# test_payment_system.py

import pytest
from payment_system import (
    CardValidator,
    FeeCalculator,
    TransactionRepository,
    NotificationService,
    PaymentProcessor
)

# Unit tests for CardValidator
def test_card_validator_valid_card():
    validator = CardValidator()
    is_valid, error = validator.validate("4532015112830366")
    assert is_valid == True
    assert error is None

def test_card_validator_invalid_card():
    validator = CardValidator()
    is_valid, error = validator.validate("1234567890123456")
    assert is_valid == False
    assert error == "Invalid card number"

# Unit tests for FeeCalculator
def test_fee_calculator_visa():
    calculator = FeeCalculator()
    fee = calculator.calculate(100.00, "visa")
    assert fee == pytest.approx(3.20)

def test_fee_calculator_amex():
    calculator = FeeCalculator()
    fee = calculator.calculate(100.00, "amex")
    assert fee == pytest.approx(3.70)

# Unit tests for TransactionRepository
def test_repository_save_transaction():
    repo = TransactionRepository()
    transaction = {"amount": 100.00, "fee": 3.20, "total": 103.20}
    
    transaction_id = repo.save(transaction)
    
    assert transaction_id == "TXN000001"
    assert len(repo.get_all()) == 1

def test_repository_get_by_id():
    repo = TransactionRepository()
    transaction = {"amount": 100.00, "fee": 3.20, "total": 103.20}
    
    transaction_id = repo.save(transaction)
    retrieved = repo.get_by_id(transaction_id)
    
    assert retrieved is not None
    assert retrieved["amount"] == 100.00

# Unit tests for NotificationService
def test_notification_service_send_receipt():
    service = NotificationService()
    transaction = {"id": "TXN000001", "amount": 100.00}
    
    service.send_receipt("customer@example.com", transaction)
    
    assert len(service.sent_notifications) == 1
    assert service.sent_notifications[0]["type"] == "receipt"
```

Running these tests:

```bash
$ pytest test_payment_system.py -v
```

**Output**:

```text
test_payment_system.py::test_card_validator_valid_card PASSED            [ 14%]
test_payment_system.py::test_card_validator_invalid_card PASSED          [ 28%]
test_payment_system.py::test_fee_calculator_visa PASSED                  [ 42%]
test_payment_system.py::test_fee_calculator_amex PASSED                  [ 57%]
test_payment_system.py::test_repository_save_transaction PASSED          [ 71%]
test_payment_system.py::test_repository_get_by_id PASSED                 [ 85%]
test_payment_system.py::test_notification_service_send_receipt PASSED    [100%]

========================== 7 passed in 0.03s ==========================
```

Good! Each component works in isolation. But do they work together?

**Current limitation**: We've only tested components independently. We haven't verified their integration.

### Iteration 1: Testing Component Integration

Let's test that the `PaymentProcessor` correctly integrates all components:

```python
# test_payment_system.py

import pytest
from payment_system import (
    CardValidator,
    FeeCalculator,
    TransactionRepository,
    NotificationService,
    PaymentProcessor
)

@pytest.fixture
def payment_system():
    """Provide a complete payment system for testing."""
    validator = CardValidator()
    calculator = FeeCalculator()
    repository = TransactionRepository()
    notifications = NotificationService()
    
    processor = PaymentProcessor(
        validator=validator,
        fee_calculator=calculator,
        repository=repository,
        notification_service=notifications
    )
    
    return {
        "processor": processor,
        "validator": validator,
        "calculator": calculator,
        "repository": repository,
        "notifications": notifications
    }

def test_successful_payment_integration(payment_system):
    """Test complete payment flow with all components."""
    processor = payment_system["processor"]
    repository = payment_system["repository"]
    
    result = processor.process_payment(
        card_number="4532015112830366",
        amount=100.00,
        card_type="visa"
    )
    
    # Verify payment succeeded
    assert result["success"] == True
    assert result["transaction_id"] == "TXN000001"
    
    # Verify transaction was saved
    saved_txn = repository.get_by_id("TXN000001")
    assert saved_txn is not None
    assert saved_txn["amount"] == 100.00
    assert saved_txn["fee"] == pytest.approx(3.20)
    
    # Verify total processed is correct
    assert processor.get_total_processed() == pytest.approx(103.20)
```

Running this test:

```bash
$ pytest test_payment_system.py::test_successful_payment_integration -v
```

**Output**:

```text
test_payment_system.py::test_successful_payment_integration PASSED       [100%]

========================== 1 passed in 0.01s ==========================
```

Excellent! The components work together correctly. But what happens when validation fails? Does the transaction still get saved?

**Current limitation**: We haven't tested error paths in the integration.

### Iteration 2: Testing Error Propagation

Let's test that errors in one component are handled correctly by the system:

```python
# test_payment_system.py

import pytest
from payment_system import (
    CardValidator,
    FeeCalculator,
    TransactionRepository,
    NotificationService,
    PaymentProcessor
)

@pytest.fixture
def payment_system():
    """Provide a complete payment system for testing."""
    validator = CardValidator()
    calculator = FeeCalculator()
    repository = TransactionRepository()
    notifications = NotificationService()
    
    processor = PaymentProcessor(
        validator=validator,
        fee_calculator=calculator,
        repository=repository,
        notification_service=notifications
    )
    
    return {
        "processor": processor,
        "validator": validator,
        "calculator": calculator,
        "repository": repository,
        "notifications": notifications
    }

def test_failed_validation_no_transaction_saved(payment_system):
    """Test that failed validation doesn't save transaction."""
    processor = payment_system["processor"]
    repository = payment_system["repository"]
    
    result = processor.process_payment(
        card_number="1234567890123456",  # Invalid card
        amount=100.00,
        card_type="visa"
    )
    
    # Verify payment failed
    assert result["success"] == False
    assert result["error"] == "Invalid card number"
    
    # Verify no transaction was saved
    assert len(repository.get_all()) == 0
    assert processor.get_total_processed() == 0.0

def test_invalid_amount_no_transaction_saved(payment_system):
    """Test that invalid amount doesn't save transaction."""
    processor = payment_system["processor"]
    repository = payment_system["repository"]
    
    result = processor.process_payment(
        card_number="4532015112830366",
        amount=-50.00,  # Invalid amount
        card_type="visa"
    )
    
    # Verify payment failed
    assert result["success"] == False
    assert result["error"] == "Amount must be positive"
    
    # Verify no transaction was saved
    assert len(repository.get_all()) == 0
```

Running these tests:

```bash
$ pytest test_payment_system.py -k "failed_validation or invalid_amount" -v
```

**Output**:

```text
test_payment_system.py::test_failed_validation_no_transaction_saved PASSED [ 50%]
test_payment_system.py::test_invalid_amount_no_transaction_saved PASSED  [100%]

========================== 2 passed in 0.02s ==========================
```

Perfect! Errors are handled correctly—failed payments don't create transactions.

**Current state**: We've verified that components integrate correctly and errors propagate properly.

### Iteration 3: Testing Notification Integration

Let's test that notifications are sent correctly:

```python
# test_payment_system.py

import pytest
from payment_system import (
    CardValidator,
    FeeCalculator,
    TransactionRepository,
    NotificationService,
    PaymentProcessor
)

@pytest.fixture
def payment_system():
    """Provide a complete payment system for testing."""
    validator = CardValidator()
    calculator = FeeCalculator()
    repository = TransactionRepository()
    notifications = NotificationService()
    
    processor = PaymentProcessor(
        validator=validator,
        fee_calculator=calculator,
        repository=repository,
        notification_service=notifications
    )
    
    return {
        "processor": processor,
        "validator": validator,
        "calculator": calculator,
        "repository": repository,
        "notifications": notifications
    }

def test_receipt_sent_with_email(payment_system):
    """Test that receipt is sent when email provided."""
    processor = payment_system["processor"]
    notifications = payment_system["notifications"]
    
    result = processor.process_payment(
        card_number="4532015112830366",
        amount=100.00,
        card_type="visa",
        customer_email="customer@example.com"
    )
    
    assert result["success"] == True
    
    # Verify receipt was sent
    assert len(notifications.sent_notifications) == 1
    notification = notifications.sent_notifications[0]
    assert notification["type"] == "receipt"
    assert notification["email"] == "customer@example.com"
    assert notification["transaction_id"] == result["transaction_id"]

def test_no_receipt_without_email(payment_system):
    """Test that no receipt is sent without email."""
    processor = payment_system["processor"]
    notifications = payment_system["notifications"]
    
    result = processor.process_payment(
        card_number="4532015112830366",
        amount=100.00,
        card_type="visa"
    )
    
    assert result["success"] == True
    
    # Verify no receipt was sent
    assert len(notifications.sent_notifications) == 0

def test_fraud_alert_for_high_value(payment_system):
    """Test that fraud alert is sent for high-value transactions."""
    processor = payment_system["processor"]
    notifications = payment_system["notifications"]
    
    result = processor.process_payment(
        card_number="4532015112830366",
        amount=15000.00,
        card_type="visa"
    )
    
    assert result["success"] == True
    
    # Verify fraud alert was sent
    fraud_alerts = [n for n in notifications.sent_notifications if n["type"] == "fraud_alert"]
    assert len(fraud_alerts) == 1
    assert fraud_alerts[0]["transaction_id"] == result["transaction_id"]
```

Running these tests:

```bash
$ pytest test_payment_system.py -k "receipt or fraud" -v
```

**Output**:

```text
test_payment_system.py::test_receipt_sent_with_email PASSED              [ 33%]
test_payment_system.py::test_no_receipt_without_email PASSED             [ 66%]
test_payment_system.py::test_fraud_alert_for_high_value PASSED           [100%]

========================== 3 passed in 0.02s ==========================
```

Excellent! Notifications are integrated correctly.

### Iteration 4: Testing Multiple Transactions

Let's test that the system handles multiple transactions correctly:

```python
# test_payment_system.py

import pytest
from payment_system import (
    CardValidator,
    FeeCalculator,
    TransactionRepository,
    NotificationService,
    PaymentProcessor
)

@pytest.fixture
def payment_system():
    """Provide a complete payment system for testing."""
    validator = CardValidator()
    calculator = FeeCalculator()
    repository = TransactionRepository()
    notifications = NotificationService()
    
    processor = PaymentProcessor(
        validator=validator,
        fee_calculator=calculator,
        repository=repository,
        notification_service=notifications
    )
    
    return {
        "processor": processor,
        "validator": validator,
        "calculator": calculator,
        "repository": repository,
        "notifications": notifications
    }

def test_multiple_transactions_accumulate(payment_system):
    """Test that multiple transactions are tracked correctly."""
    processor = payment_system["processor"]
    repository = payment_system["repository"]
    
    # Process three transactions
    result1 = processor.process_payment("4532015112830366", 100.00, "visa")
    result2 = processor.process_payment("4532015112830366", 50.00, "visa")
    result3 = processor.process_payment("378282246310005", 200.00, "amex")
    
    # Verify all succeeded
    assert result1["success"] == True
    assert result2["success"] == True
    assert result3["success"] == True
    
    # Verify transaction IDs are sequential
    assert result1["transaction_id"] == "TXN000001"
    assert result2["transaction_id"] == "TXN000002"
    assert result3["transaction_id"] == "TXN000003"
    
    # Verify all transactions are saved
    assert len(repository.get_all()) == 3
    
    # Verify total processed is correct
    # $103.20 + $51.75 + $207.40 = $362.35
    expected_total = 103.20 + 51.75 + 207.40
    assert processor.get_total_processed() == pytest.approx(expected_total)

def test_mixed_success_and_failure(payment_system):
    """Test that failed transactions don't affect successful ones."""
    processor = payment_system["processor"]
    repository = payment_system["repository"]
    
    # Process mix of valid and invalid transactions
    result1 = processor.process_payment("4532015112830366", 100.00, "visa")
    result2 = processor.process_payment("1234567890123456", 50.00, "visa")  # Invalid
    result3 = processor.process_payment("4532015112830366", 75.00, "visa")
    
    # Verify results
    assert result1["success"] == True
    assert result2["success"] == False
    assert result3["success"] == True
    
    # Verify only successful transactions are saved
    assert len(repository.get_all()) == 2
    
    # Verify transaction IDs skip failed transaction
    assert result1["transaction_id"] == "TXN000001"
    assert result3["transaction_id"] == "TXN000002"
    
    # Verify total only includes successful transactions
    expected_total = 103.20 + 77.45  # $100 + $3.20 + $75 + $2.45
    assert processor.get_total_processed() == pytest.approx(expected_total)
```

Running these tests:

```bash
$ pytest test_payment_system.py -k "multiple or mixed" -v
```

**Output**:

```text
test_payment_system.py::test_multiple_transactions_accumulate PASSED     [ 50%]
test_payment_system.py::test_mixed_success_and_failure PASSED            [100%]

========================== 2 passed in 0.02s ==========================
```

Perfect! The system handles multiple transactions correctly, including mixed success and failure scenarios.

### The Journey: From Problem to Solution

| Iteration | Problem                          | Solution Applied                | Result                    |
|-----------|----------------------------------|---------------------------------|---------------------------|
| 0         | Components not tested together   | Test components in isolation    | Unit tests pass           |
| 1         | Integration not verified         | Test complete payment flow      | Integration verified      |
| 2         | Error handling not tested        | Test error propagation          | Errors handled correctly  |
| 3         | Notifications not verified       | Test notification integration   | Notifications work        |
| 4         | Multiple transactions not tested | Test transaction accumulation   | System scales correctly   |

### Final Implementation

Here's our complete integration test suite:

```python
# test_payment_system.py

import pytest
from payment_system import (
    CardValidator,
    FeeCalculator,
    TransactionRepository,
    NotificationService,
    PaymentProcessor
)

@pytest.fixture
def payment_system():
    """Provide a complete payment system for testing."""
    validator = CardValidator()
    calculator = FeeCalculator()
    repository = TransactionRepository()
    notifications = NotificationService()
    
    processor = PaymentProcessor(
        validator=validator,
        fee_calculator=calculator,
        repository=repository,
        notification_service=notifications
    )
    
    return {
        "processor": processor,
        "validator": validator,
        "calculator": calculator,
        "repository": repository,
        "notifications": notifications
    }

# Integration tests
def test_successful_payment_integration(payment_system):
    """Test complete payment flow with all components."""
    processor = payment_system["processor"]
    repository = payment_system["repository"]
    
    result = processor.process_payment(
        card_number="4532015112830366",
        amount=100.00,
        card_type="visa"
    )
    
    assert result["success"] == True
    assert result["transaction_id"] == "TXN000001"
    
    saved_txn = repository.get_by_id("TXN000001")
    assert saved_txn is not None
    assert saved_txn["amount"] == 100.00
    assert processor.get_total_processed() == pytest.approx(103.20)

def test_failed_validation_no_transaction_saved(payment_system):
    """Test that failed validation doesn't save transaction."""
    processor = payment_system["processor"]
    repository = payment_system["repository"]
    
    result = processor.process_payment(
        card_number="1234567890123456",
        amount=100.00,
        card_type="visa"
    )
    
    assert result["success"] == False
    assert len(repository.get_all()) == 0

def test_receipt_sent_with_email(payment_system):
    """Test that receipt is sent when email provided."""
    processor = payment_system["processor"]
    notifications = payment_system["notifications"]
    
    result = processor.process_payment(
        card_number="4532015112830366",
        amount=100.00,
        card_type="visa",
        customer_email="customer@example.com"
    )
    
    assert result["success"] == True
    assert len(notifications.sent_notifications) == 1
    assert notifications.sent_notifications[0]["type"] == "receipt"

def test_fraud_alert_for_high_value(payment_system):
    """Test that fraud alert is sent for high-value transactions."""
    processor = payment_system["processor"]
    notifications = payment_system["notifications"]
    
    result = processor.process_payment(
        card_number="4532015112830366",
        amount=15000.00,
        card_type="visa"
    )
    
    assert result["success"] == True
    fraud_alerts = [n for n in notifications.sent_notifications if n["type"] == "fraud_alert"]
    assert len(fraud_alerts) == 1

def test_multiple_transactions_accumulate(payment_system):
    """Test that multiple transactions are tracked correctly."""
    processor = payment_system["processor"]
    repository = payment_system["repository"]
    
    processor.process_payment("4532015112830366", 100.00, "visa")
    processor.process_payment("4532015112830366", 50.00, "visa")
    processor.process_payment("378282246310005", 200.00, "amex")
    
    assert len(repository.get_all()) == 3
    expected_total = 103.20 + 51.75 + 207.40
    assert processor.get_total_processed() == pytest.approx(expected_total)

def test_mixed_success_and_failure(payment_system):
    """Test that failed transactions don't affect successful ones."""
    processor = payment_system["processor"]
    repository = payment_system["repository"]
    
    result1 = processor.process_payment("4532015112830366", 100.00, "visa")
    result2 = processor.process_payment("1234567890123456", 50.00, "visa")
    result3 = processor.process_payment("4532015112830366", 75.00, "visa")
    
    assert result1["success"] == True
    assert result2["success"] == False
    assert result3["success"] == True
    assert len(repository.get_all()) == 2
```

### Decision Framework: Unit vs. Integration Tests

**Unit tests**:
- Test components in isolation
- Fast execution
- Pinpoint failures to specific components
- Use mocks for dependencies

**Integration tests**:
- Test components working together
- Slower execution
- Verify component interactions
- Use real implementations

**For our payment system**:
- **Unit tests**: Test each component (CardValidator, FeeCalculator, etc.) independently
- **Integration tests**: Test PaymentProcessor with all real components
- **Balance**: More unit tests (fast feedback), fewer integration tests (verify integration)

### When to Write Integration Tests

**Write integration tests when**:
- Components have complex interactions
- Data flows through multiple components
- Error handling crosses component boundaries
- State is shared between components
- You need to verify the complete user flow

**Don't write integration tests when**:
- Components are completely independent
- Unit tests provide sufficient coverage
- Integration would be too slow or complex
- The integration is trivial (simple delegation)

### Lessons Learned

1. **Test components in isolation first**: Verify each piece works before testing integration
2. **Use fixtures for system setup**: Create complete systems with all dependencies
3. **Test error propagation**: Verify errors in one component are handled correctly by others
4. **Test state accumulation**: Verify that shared state is managed correctly
5. **Test both success and failure paths**: Integration tests should cover error scenarios
6. **Keep integration tests focused**: Test specific integration points, not everything
7. **Balance unit and integration tests**: More unit tests, fewer integration tests
8. **Use real implementations**: Integration tests should use real components, not mocks
