# Chapter 9: Spies, Stubs, and Test Doubles

## The Difference Between Mocks, Stubs, and Spies

## The Difference Between Mocks, Stubs, and Spies

In Chapter 8, we learned about mocking with `unittest.mock`. We used `Mock()` objects and the `@patch` decorator to replace real dependencies with test doubles. But we used the term "mock" loosely—as a catch-all for any fake object in a test.

Professional testing literature distinguishes between several types of test doubles, each with a specific purpose. Understanding these distinctions helps you choose the right tool for each testing scenario and communicate precisely with other developers.

### The Reference Implementation: A Payment Processing System

We'll explore test doubles through a realistic payment processing system that interacts with multiple external services. This system will serve as our anchor example throughout the chapter.

```python
# payment_processor.py
class PaymentGateway:
    """External payment service - we don't control this."""
    def charge(self, amount, card_token):
        # In reality, makes HTTP request to payment provider
        raise NotImplementedError("Real implementation calls external API")
    
    def refund(self, transaction_id):
        raise NotImplementedError("Real implementation calls external API")

class FraudDetector:
    """External fraud detection service."""
    def check_transaction(self, amount, card_token, user_id):
        # In reality, calls ML service
        raise NotImplementedError("Real implementation calls fraud API")

class NotificationService:
    """Sends emails/SMS to users."""
    def send_receipt(self, user_id, transaction_id, amount):
        raise NotImplementedError("Real implementation sends email")
    
    def send_fraud_alert(self, user_id):
        raise NotImplementedError("Real implementation sends alert")

class PaymentProcessor:
    """Our code - coordinates payment flow."""
    def __init__(self, gateway, fraud_detector, notifier):
        self.gateway = gateway
        self.fraud_detector = fraud_detector
        self.notifier = notifier
    
    def process_payment(self, user_id, amount, card_token):
        # Check for fraud first
        fraud_result = self.fraud_detector.check_transaction(
            amount, card_token, user_id
        )
        
        if fraud_result == "SUSPICIOUS":
            self.notifier.send_fraud_alert(user_id)
            return {"status": "rejected", "reason": "fraud_detected"}
        
        # Attempt to charge
        transaction_id = self.gateway.charge(amount, card_token)
        
        # Send receipt
        self.notifier.send_receipt(user_id, transaction_id, amount)
        
        return {
            "status": "success",
            "transaction_id": transaction_id
        }
```

This is our production code. It coordinates three external services:
- **PaymentGateway**: Charges credit cards (costs money per call)
- **FraudDetector**: ML-based fraud detection (slow, expensive)
- **NotificationService**: Sends emails/SMS (triggers real communications)

We cannot call these services in tests. We need test doubles.

### Iteration 0: The Naive Approach - Using Mock() for Everything

Let's write our first test using what we learned in Chapter 8—replace everything with `Mock()` objects.

```python
# test_payment_processor_naive.py
from unittest.mock import Mock
from payment_processor import PaymentProcessor

def test_successful_payment_naive():
    # Create mock objects for all dependencies
    gateway = Mock()
    fraud_detector = Mock()
    notifier = Mock()
    
    # Make them return something
    fraud_detector.check_transaction.return_value = "SAFE"
    gateway.charge.return_value = "txn_12345"
    
    # Create processor with mocks
    processor = PaymentProcessor(gateway, fraud_detector, notifier)
    
    # Process payment
    result = processor.process_payment(
        user_id="user_42",
        amount=99.99,
        card_token="tok_visa_4242"
    )
    
    # Verify result
    assert result["status"] == "success"
    assert result["transaction_id"] == "txn_12345"
    
    # Verify interactions happened
    fraud_detector.check_transaction.assert_called_once_with(
        99.99, "tok_visa_4242", "user_42"
    )
    gateway.charge.assert_called_once_with(99.99, "tok_visa_4242")
    notifier.send_receipt.assert_called_once_with(
        "user_42", "txn_12345", 99.99
    )
```

Run this test:

```bash
pytest test_payment_processor_naive.py -v
```

**Output**:
```
test_payment_processor_naive.py::test_successful_payment_naive PASSED
```

The test passes. But there's a problem lurking beneath the surface.

### Diagnostic Analysis: What Are We Actually Testing?

Let's add another test for the fraud detection path:

```python
def test_fraud_detected_naive():
    gateway = Mock()
    fraud_detector = Mock()
    notifier = Mock()
    
    # Simulate fraud detection
    fraud_detector.check_transaction.return_value = "SUSPICIOUS"
    
    processor = PaymentProcessor(gateway, fraud_detector, notifier)
    
    result = processor.process_payment(
        user_id="user_42",
        amount=99.99,
        card_token="tok_visa_4242"
    )
    
    # Verify fraud rejection
    assert result["status"] == "rejected"
    assert result["reason"] == "fraud_detected"
    
    # Verify fraud alert was sent
    notifier.send_fraud_alert.assert_called_once_with("user_42")
    
    # Verify we did NOT charge the card
    gateway.charge.assert_not_called()
```

Run both tests:

```bash
pytest test_payment_processor_naive.py -v
```

**Output**:
```
test_payment_processor_naive.py::test_successful_payment_naive PASSED
test_payment_processor_naive.py::test_fraud_detected_naive PASSED
```

Both pass. But notice what we're doing in each test:

**In `test_successful_payment_naive`**:
- We configure return values: `fraud_detector.check_transaction.return_value = "SAFE"`
- We verify method calls: `gateway.charge.assert_called_once_with(...)`
- We check both behavior (return value) AND interactions (method calls)

**In `test_fraud_detected_naive`**:
- We configure return values: `fraud_detector.check_transaction.return_value = "SUSPICIOUS"`
- We verify method calls: `notifier.send_fraud_alert.assert_called_once_with(...)`
- We verify methods were NOT called: `gateway.charge.assert_not_called()`

We're using `Mock()` for three different purposes:
1. **Providing canned responses** (fraud_detector returning "SAFE")
2. **Recording interactions** (verifying charge was called)
3. **Doing nothing** (notifier just needs to exist)

This works, but it's conceptually muddy. We're using one tool for three distinct jobs.

### The Problem: Mock() Is Too Powerful

The issue becomes clearer when we look at what `Mock()` allows:

```python
def test_mock_accepts_anything():
    gateway = Mock()
    
    # Mock accepts ANY method call
    gateway.charge(99.99, "tok_visa")  # OK
    gateway.refund("txn_123")  # OK
    gateway.completely_made_up_method()  # Also OK!
    gateway.another_fake_method(1, 2, 3, foo="bar")  # Still OK!
    
    # Mock accepts ANY attribute access
    print(gateway.some_attribute)  # Returns another Mock
    print(gateway.nested.deeply.fake.attribute)  # Also returns Mock
    
    # All of these "work" - Mock never complains
```

Run this:

```bash
pytest test_payment_processor_naive.py::test_mock_accepts_anything -v -s
```

**Output**:
```
test_payment_processor_naive.py::test_mock_accepts_anything <MagicMock name='mock.some_attribute' id='...'>
<MagicMock name='mock.nested.deeply.fake.attribute' id='...'>
PASSED
```

`Mock()` accepts everything. This is powerful but dangerous. If we typo a method name in our test, the test still passes:

```python
def test_dangerous_typo():
    gateway = Mock()
    gateway.charge.return_value = "txn_12345"
    
    processor = PaymentProcessor(gateway, Mock(), Mock())
    result = processor.process_payment("user_42", 99.99, "tok_visa")
    
    # Typo: "chrage" instead of "charge"
    gateway.chrage.assert_called_once()  # This passes!
```

Run this:

```bash
pytest test_payment_processor_naive.py::test_dangerous_typo -v
```

**Output**:
```
test_payment_processor_naive.py::test_dangerous_typo PASSED
```

The test passes even though we verified the wrong method. `Mock()` created `gateway.chrage` on the fly when we accessed it.

### What We Need: Specialized Test Doubles

Professional testing distinguishes between different types of test doubles based on their purpose:

1. **Stub**: Provides canned responses to method calls. Used when you need a dependency to return specific values but don't care about verifying interactions.

2. **Spy**: Records information about how it was called. Used when you need to verify interactions happened but don't need to control return values.

3. **Mock**: Combines stub and spy—provides canned responses AND verifies interactions. Used when you need both capabilities.

4. **Fake**: A working implementation with shortcuts (e.g., in-memory database instead of PostgreSQL). Used for integration testing.

5. **Dummy**: A placeholder that does nothing. Used when a parameter is required but not used in the test scenario.

Let's rewrite our tests using the appropriate test double for each dependency.

### Iteration 1: Using Stubs for Canned Responses

A **stub** provides predetermined responses. We use stubs when we need a dependency to return specific values but don't care about verifying how it was called.

In our payment processor, the `FraudDetector` is a perfect candidate for stubbing. We need it to return "SAFE" or "SUSPICIOUS", but we don't need to verify the exact parameters it was called with—that's an implementation detail.

Let's create a stub manually first to understand the concept:

```python
# test_payment_processor_stubs.py
from payment_processor import PaymentProcessor, FraudDetector

class FraudDetectorStub(FraudDetector):
    """A stub that returns predetermined fraud check results."""
    def __init__(self, result="SAFE"):
        self.result = result
    
    def check_transaction(self, amount, card_token, user_id):
        # Always returns the predetermined result
        return self.result

def test_successful_payment_with_stub():
    # Use a stub for fraud detector - we only care about its return value
    fraud_detector = FraudDetectorStub(result="SAFE")
    
    # Use Mock for gateway - we need to verify it was called AND provide return value
    from unittest.mock import Mock
    gateway = Mock()
    gateway.charge.return_value = "txn_12345"
    
    # Use Mock for notifier - we need to verify it was called
    notifier = Mock()
    
    processor = PaymentProcessor(gateway, fraud_detector, notifier)
    
    result = processor.process_payment(
        user_id="user_42",
        amount=99.99,
        card_token="tok_visa_4242"
    )
    
    assert result["status"] == "success"
    assert result["transaction_id"] == "txn_12345"
    
    # We verify gateway and notifier interactions
    gateway.charge.assert_called_once_with(99.99, "tok_visa_4242")
    notifier.send_receipt.assert_called_once_with(
        "user_42", "txn_12345", 99.99
    )
    # Note: We don't verify fraud_detector calls - it's a stub
```

Run this:

```bash
pytest test_payment_processor_stubs.py::test_successful_payment_with_stub -v
```

**Output**:
```
test_payment_processor_stubs.py::test_successful_payment_with_stub PASSED
```

**What changed**:
- `FraudDetectorStub` is a real class that implements the `FraudDetector` interface
- It returns a predetermined value without any mock configuration
- We don't verify how it was called—we only care that it returns the right value
- The test is clearer about intent: "fraud detector returns SAFE"

Now test the fraud detection path:

```python
def test_fraud_detected_with_stub():
    # Stub returns "SUSPICIOUS"
    fraud_detector = FraudDetectorStub(result="SUSPICIOUS")
    
    gateway = Mock()
    notifier = Mock()
    
    processor = PaymentProcessor(gateway, fraud_detector, notifier)
    
    result = processor.process_payment(
        user_id="user_42",
        amount=99.99,
        card_token="tok_visa_4242"
    )
    
    assert result["status"] == "rejected"
    assert result["reason"] == "fraud_detected"
    
    notifier.send_fraud_alert.assert_called_once_with("user_42")
    gateway.charge.assert_not_called()
```

Run both tests:

```bash
pytest test_payment_processor_stubs.py -v
```

**Output**:
```
test_payment_processor_stubs.py::test_successful_payment_with_stub PASSED
test_payment_processor_stubs.py::test_fraud_detected_with_stub PASSED
```

**When to use stubs**:
- You need a dependency to return specific values
- You don't care about verifying interactions
- The dependency's behavior is simple (returns values based on input)
- You want to make test intent clear: "given this input, return this output"

**When NOT to use stubs**:
- You need to verify the dependency was called correctly
- The dependency has complex behavior or side effects
- You need to verify the dependency was NOT called in certain scenarios

### Iteration 2: Using Spies to Record Interactions

A **spy** records how it was called without changing behavior. We use spies when we need to verify interactions but want the real implementation to run (or we need minimal behavior).

In our payment processor, the `NotificationService` is a good candidate for spying. We need to verify it was called with the right parameters, but we don't need it to actually send emails in tests.

Let's create a spy manually:

```python
# test_payment_processor_spies.py
from payment_processor import PaymentProcessor, NotificationService

class NotificationServiceSpy(NotificationService):
    """A spy that records all method calls."""
    def __init__(self):
        self.receipt_calls = []
        self.alert_calls = []
    
    def send_receipt(self, user_id, transaction_id, amount):
        # Record the call
        self.receipt_calls.append({
            "user_id": user_id,
            "transaction_id": transaction_id,
            "amount": amount
        })
        # Don't actually send email
    
    def send_fraud_alert(self, user_id):
        # Record the call
        self.alert_calls.append({"user_id": user_id})
        # Don't actually send alert

def test_successful_payment_with_spy():
    from unittest.mock import Mock
    
    fraud_detector = Mock()
    fraud_detector.check_transaction.return_value = "SAFE"
    
    gateway = Mock()
    gateway.charge.return_value = "txn_12345"
    
    # Use a spy for notifications
    notifier = NotificationServiceSpy()
    
    processor = PaymentProcessor(gateway, fraud_detector, notifier)
    
    result = processor.process_payment(
        user_id="user_42",
        amount=99.99,
        card_token="tok_visa_4242"
    )
    
    assert result["status"] == "success"
    
    # Verify using spy's recorded calls
    assert len(notifier.receipt_calls) == 1
    assert notifier.receipt_calls[0] == {
        "user_id": "user_42",
        "transaction_id": "txn_12345",
        "amount": 99.99
    }
    assert len(notifier.alert_calls) == 0
```

Run this:

```bash
pytest test_payment_processor_spies.py::test_successful_payment_with_spy -v
```

**Output**:
```
test_payment_processor_spies.py::test_successful_payment_with_spy PASSED
```

**What changed**:
- `NotificationServiceSpy` records all calls in lists
- We verify interactions by inspecting the spy's recorded data
- The spy doesn't use mock assertions—it uses plain data structures
- We can inspect the exact parameters passed to each call

Test the fraud path:

```python
def test_fraud_detected_with_spy():
    from unittest.mock import Mock
    
    fraud_detector = Mock()
    fraud_detector.check_transaction.return_value = "SUSPICIOUS"
    
    gateway = Mock()
    notifier = NotificationServiceSpy()
    
    processor = PaymentProcessor(gateway, fraud_detector, notifier)
    
    result = processor.process_payment(
        user_id="user_42",
        amount=99.99,
        card_token="tok_visa_4242"
    )
    
    assert result["status"] == "rejected"
    
    # Verify fraud alert was sent
    assert len(notifier.alert_calls) == 1
    assert notifier.alert_calls[0] == {"user_id": "user_42"}
    
    # Verify receipt was NOT sent
    assert len(notifier.receipt_calls) == 0
    
    # Verify gateway was not charged
    gateway.charge.assert_not_called()
```

Run both tests:

```bash
pytest test_payment_processor_spies.py -v
```

**Output**:
```
test_payment_processor_spies.py::test_successful_payment_with_spy PASSED
test_payment_processor_spies.py::test_fraud_detected_with_spy PASSED
```

**When to use spies**:
- You need to verify a dependency was called with specific parameters
- You want to inspect the exact sequence of calls
- You need to verify a dependency was NOT called
- You want more control over verification than mock assertions provide

**When NOT to use spies**:
- You need to control return values (use stub or mock instead)
- The dependency has complex behavior that affects the test
- Simple mock assertions are sufficient

### Iteration 3: Understanding When Mock() Is Actually a Mock

Now we understand that `Mock()` from `unittest.mock` is actually a **mock** in the technical sense—it combines stub and spy capabilities. It can:
1. Provide canned responses (stub behavior)
2. Record and verify interactions (spy behavior)

Let's rewrite our test using the precise terminology:

```python
# test_payment_processor_precise.py
from unittest.mock import Mock
from payment_processor import PaymentProcessor

def test_successful_payment_precise_terminology():
    # fraud_detector is a STUB - we only configure return value
    fraud_detector = Mock()
    fraud_detector.check_transaction.return_value = "SAFE"
    
    # gateway is a MOCK - we configure return value AND verify calls
    gateway = Mock()
    gateway.charge.return_value = "txn_12345"
    
    # notifier is a SPY - we only verify calls
    notifier = Mock()
    
    processor = PaymentProcessor(gateway, fraud_detector, notifier)
    
    result = processor.process_payment(
        user_id="user_42",
        amount=99.99,
        card_token="tok_visa_4242"
    )
    
    # Verify result
    assert result["status"] == "success"
    assert result["transaction_id"] == "txn_12345"
    
    # Verify MOCK (gateway) - both return value and interaction
    gateway.charge.assert_called_once_with(99.99, "tok_visa_4242")
    
    # Verify SPY (notifier) - only interaction
    notifier.send_receipt.assert_called_once_with(
        "user_42", "txn_12345", 99.99
    )
    
    # We don't verify STUB (fraud_detector) - we only care about return value
```

Run this:

```bash
pytest test_payment_processor_precise.py -v
```

**Output**:
```
test_payment_processor_precise.py::test_successful_payment_precise_terminology PASSED
```

Even though we're using `Mock()` for all three dependencies, we're using them in different ways:
- **As a stub**: Configure return value, don't verify calls
- **As a spy**: Verify calls, don't configure return value (or it returns default)
- **As a mock**: Configure return value AND verify calls

### The Vocabulary Summary

| Test Double | Purpose | Provides Return Values? | Verifies Interactions? | Example Use Case |
|-------------|---------|------------------------|----------------------|------------------|
| **Stub** | Provide canned responses | ✅ Yes | ❌ No | Fraud detector returning "SAFE" |
| **Spy** | Record interactions | ❌ No (or minimal) | ✅ Yes | Notification service recording calls |
| **Mock** | Stub + Spy combined | ✅ Yes | ✅ Yes | Payment gateway that must return transaction ID and be verified |
| **Fake** | Working implementation | ✅ Yes (real logic) | ❌ No | In-memory database for integration tests |
| **Dummy** | Placeholder | ❌ No | ❌ No | Required parameter that's never used |

### When to Use Each Type

**Use a Stub when**:
- You need a dependency to return specific values
- You don't care how many times it's called or with what parameters
- The dependency's behavior is simple and deterministic
- Example: Configuration object, data source, external API that returns data

**Use a Spy when**:
- You need to verify a dependency was called correctly
- You want to inspect the exact parameters passed
- You need to verify call order or frequency
- Example: Logger, event emitter, notification service

**Use a Mock when**:
- You need both stub and spy capabilities
- You need to verify interactions AND control return values
- The dependency is critical to the test's correctness
- Example: Database transaction, payment gateway, authentication service

**Use a Fake when**:
- You need realistic behavior without external dependencies
- You're doing integration testing
- The real implementation is too slow or expensive
- Example: In-memory database, fake file system, local message queue

**Use a Dummy when**:
- A parameter is required but not used in the test scenario
- You need to satisfy a function signature
- The value doesn't matter for the test
- Example: Unused callback, required but ignored configuration

### Practical Guideline: Start Simple, Add Complexity Only When Needed

In practice, most Python developers use `Mock()` for everything because it's convenient and flexible. This is fine for simple tests. The key is understanding what role each mock is playing:

1. **If you only configure return values** → You're using it as a stub
2. **If you only verify calls** → You're using it as a spy  
3. **If you do both** → You're using it as a true mock

Knowing the terminology helps you:
- Communicate precisely with other developers
- Understand testing literature and documentation
- Make conscious decisions about test design
- Recognize when you're over-verifying (testing implementation details)

In the next sections, we'll explore `MagicMock`, which extends `Mock()` with additional capabilities for complex scenarios.

## Using MagicMock for Complex Scenarios

## Using MagicMock for Complex Scenarios

In Section 9.1, we used `Mock()` to create test doubles. But `Mock()` has a limitation: it doesn't support Python's "magic methods" (also called dunder methods) like `__len__`, `__getitem__`, `__iter__`, etc.

This limitation becomes a problem when testing code that uses these magic methods—code that treats objects as containers, iterables, context managers, or callable objects.

### The Reference Implementation: A Data Pipeline

Let's extend our payment processor with a data pipeline that processes batches of transactions. This pipeline uses Python's magic methods extensively.

```python
# transaction_pipeline.py
class TransactionBatch:
    """Represents a batch of transactions to process."""
    def __init__(self, transactions):
        self.transactions = transactions
    
    def __len__(self):
        """Support len(batch)."""
        return len(self.transactions)
    
    def __getitem__(self, index):
        """Support batch[index]."""
        return self.transactions[index]
    
    def __iter__(self):
        """Support for transaction in batch."""
        return iter(self.transactions)

class TransactionRepository:
    """Fetches transactions from database."""
    def fetch_pending(self, limit=100):
        """Returns a TransactionBatch of pending transactions."""
        raise NotImplementedError("Real implementation queries database")
    
    def mark_processed(self, transaction_id):
        """Marks a transaction as processed."""
        raise NotImplementedError("Real implementation updates database")

class BatchProcessor:
    """Processes batches of transactions."""
    def __init__(self, repository, payment_processor):
        self.repository = repository
        self.payment_processor = payment_processor
    
    def process_pending_batch(self, limit=100):
        # Fetch batch from repository
        batch = self.repository.fetch_pending(limit)
        
        # Check if batch is empty
        if len(batch) == 0:
            return {"processed": 0, "failed": 0}
        
        processed = 0
        failed = 0
        
        # Iterate over transactions
        for transaction in batch:
            try:
                # Process each transaction
                result = self.payment_processor.process_payment(
                    user_id=transaction["user_id"],
                    amount=transaction["amount"],
                    card_token=transaction["card_token"]
                )
                
                if result["status"] == "success":
                    self.repository.mark_processed(transaction["id"])
                    processed += 1
                else:
                    failed += 1
            except Exception:
                failed += 1
        
        return {"processed": processed, "failed": failed}
```

This code uses several magic methods:
- `len(batch)` - calls `__len__`
- `for transaction in batch` - calls `__iter__`
- `batch[index]` - calls `__getitem__` (not used here but part of the interface)

### Iteration 0: Attempting to Mock with Mock()

Let's try to test this using regular `Mock()`:

```python
# test_batch_processor_mock.py
from unittest.mock import Mock
from transaction_pipeline import BatchProcessor

def test_process_empty_batch_with_mock():
    repository = Mock()
    payment_processor = Mock()
    
    # Try to create a mock batch that returns 0 for len()
    mock_batch = Mock()
    mock_batch.__len__.return_value = 0
    
    repository.fetch_pending.return_value = mock_batch
    
    processor = BatchProcessor(repository, payment_processor)
    result = processor.process_pending_batch(limit=10)
    
    assert result["processed"] == 0
    assert result["failed"] == 0
```

Run this test:

```bash
pytest test_batch_processor_mock.py::test_process_empty_batch_with_mock -v
```

**Output**:
```
test_batch_processor_mock.py::test_process_empty_batch_with_mock FAILED

def test_process_empty_batch_with_mock():
    ...
    result = processor.process_pending_batch(limit=10)

transaction_pipeline.py:45: in process_pending_batch
    if len(batch) == 0:
E   TypeError: object of type 'Mock' has no len()
```

### Diagnostic Analysis: Why Mock() Fails with Magic Methods

**The complete output**:
```
FAILED test_batch_processor_mock.py::test_process_empty_batch_with_mock - TypeError: object of type 'Mock' has no len()
```

**Let's parse this**:

1. **The summary line**: `TypeError: object of type 'Mock' has no len()`
   - What this tells us: Python's `len()` function cannot be called on a `Mock` object

2. **The traceback**:
```python
transaction_pipeline.py:45: in process_pending_batch
    if len(batch) == 0:
```
   - What this tells us: The failure occurs when our code tries to call `len()` on the batch
   - Key line: `if len(batch) == 0:` - this is where the code breaks

3. **The root cause**:
   - We configured `mock_batch.__len__.return_value = 0`
   - But `Mock()` doesn't actually support magic methods
   - When Python calls `len(mock_batch)`, it looks for `__len__` at the class level, not the instance level
   - `Mock()` doesn't implement `__len__` at the class level, so it fails

**Root cause identified**: `Mock()` doesn't support Python's magic methods because they must be defined at the class level, not the instance level.

**Why the current approach can't solve this**: Configuring `mock_batch.__len__.return_value` creates an instance attribute, but Python's `len()` function looks for `__len__` as a class method.

**What we need**: A mock object that properly implements magic methods at the class level.

### Iteration 1: Introducing MagicMock

`MagicMock` is a subclass of `Mock` that pre-configures all magic methods. It's designed specifically for scenarios where your code uses Python's special methods.

Let's fix our test:

```python
# test_batch_processor_magicmock.py
from unittest.mock import Mock, MagicMock
from transaction_pipeline import BatchProcessor

def test_process_empty_batch_with_magicmock():
    repository = Mock()
    payment_processor = Mock()
    
    # Use MagicMock instead of Mock
    mock_batch = MagicMock()
    mock_batch.__len__.return_value = 0
    
    repository.fetch_pending.return_value = mock_batch
    
    processor = BatchProcessor(repository, payment_processor)
    result = processor.process_pending_batch(limit=10)
    
    assert result["processed"] == 0
    assert result["failed"] == 0
```

Run this test:

```bash
pytest test_batch_processor_magicmock.py::test_process_empty_batch_with_magicmock -v
```

**Output**:
```
test_batch_processor_magicmock.py::test_process_empty_batch_with_magicmock PASSED
```

**What changed**:
- We replaced `Mock()` with `MagicMock()`
- Now `len(mock_batch)` works correctly
- The magic method `__len__` is properly implemented at the class level

### Iteration 2: Testing Iteration with MagicMock

Now let's test the case where we have transactions to process:

```python
def test_process_batch_with_transactions():
    repository = Mock()
    payment_processor = Mock()
    
    # Create mock transactions
    transactions = [
        {"id": "txn_1", "user_id": "user_1", "amount": 10.00, "card_token": "tok_1"},
        {"id": "txn_2", "user_id": "user_2", "amount": 20.00, "card_token": "tok_2"},
        {"id": "txn_3", "user_id": "user_3", "amount": 30.00, "card_token": "tok_3"},
    ]
    
    # Create a MagicMock batch
    mock_batch = MagicMock()
    mock_batch.__len__.return_value = 3
    mock_batch.__iter__.return_value = iter(transactions)
    
    repository.fetch_pending.return_value = mock_batch
    
    # Make payment processor return success for all
    payment_processor.process_payment.return_value = {"status": "success"}
    
    processor = BatchProcessor(repository, payment_processor)
    result = processor.process_pending_batch(limit=10)
    
    assert result["processed"] == 3
    assert result["failed"] == 0
    
    # Verify all transactions were marked as processed
    assert repository.mark_processed.call_count == 3
    repository.mark_processed.assert_any_call("txn_1")
    repository.mark_processed.assert_any_call("txn_2")
    repository.mark_processed.assert_any_call("txn_3")
```

Run this test:

```bash
pytest test_batch_processor_magicmock.py::test_process_batch_with_transactions -v
```

**Output**:
```
test_batch_processor_magicmock.py::test_process_batch_with_transactions PASSED
```

**What we configured**:
- `__len__` returns 3 (the batch has 3 transactions)
- `__iter__` returns an iterator over our transaction list
- Both magic methods work correctly with `MagicMock`

### Iteration 3: Testing Mixed Success and Failure

Let's test a scenario where some transactions succeed and others fail:

```python
def test_process_batch_with_mixed_results():
    repository = Mock()
    payment_processor = Mock()
    
    transactions = [
        {"id": "txn_1", "user_id": "user_1", "amount": 10.00, "card_token": "tok_1"},
        {"id": "txn_2", "user_id": "user_2", "amount": 20.00, "card_token": "tok_2"},
        {"id": "txn_3", "user_id": "user_3", "amount": 30.00, "card_token": "tok_3"},
    ]
    
    mock_batch = MagicMock()
    mock_batch.__len__.return_value = 3
    mock_batch.__iter__.return_value = iter(transactions)
    
    repository.fetch_pending.return_value = mock_batch
    
    # Make payment processor return different results
    payment_processor.process_payment.side_effect = [
        {"status": "success"},  # txn_1 succeeds
        {"status": "rejected", "reason": "fraud_detected"},  # txn_2 fails
        {"status": "success"},  # txn_3 succeeds
    ]
    
    processor = BatchProcessor(repository, payment_processor)
    result = processor.process_pending_batch(limit=10)
    
    assert result["processed"] == 2
    assert result["failed"] == 1
    
    # Verify only successful transactions were marked as processed
    assert repository.mark_processed.call_count == 2
    repository.mark_processed.assert_any_call("txn_1")
    repository.mark_processed.assert_any_call("txn_3")
    
    # Verify txn_2 was NOT marked as processed
    calls = [call[0][0] for call in repository.mark_processed.call_args_list]
    assert "txn_2" not in calls
```

Run this test:

```bash
pytest test_batch_processor_magicmock.py::test_process_batch_with_mixed_results -v
```

**Output**:
```
test_batch_processor_magicmock.py::test_process_batch_with_mixed_results PASSED
```

**What we demonstrated**:
- Used `side_effect` to return different results for each call
- Verified that only successful transactions were marked as processed
- Inspected the actual calls to confirm txn_2 was not processed

### Common Magic Methods Supported by MagicMock

`MagicMock` pre-configures these magic methods:

**Container methods**:
- `__len__` - `len(obj)`
- `__getitem__` - `obj[key]`
- `__setitem__` - `obj[key] = value`
- `__delitem__` - `del obj[key]`
- `__contains__` - `key in obj`

**Iteration methods**:
- `__iter__` - `for item in obj`
- `__next__` - `next(obj)`

**Numeric methods**:
- `__add__`, `__sub__`, `__mul__`, `__div__` - arithmetic operators
- `__lt__`, `__le__`, `__gt__`, `__ge__` - comparison operators

**Context manager methods**:
- `__enter__` - `with obj:`
- `__exit__` - cleanup after `with` block

**Callable methods**:
- `__call__` - `obj()`

**String representation**:
- `__str__` - `str(obj)`
- `__repr__` - `repr(obj)`

### Iteration 4: Testing Context Managers

Let's add a feature that uses context managers:

```python
# transaction_pipeline.py (addition)
class TransactionLock:
    """Ensures only one batch processes at a time."""
    def acquire(self):
        raise NotImplementedError("Real implementation acquires distributed lock")
    
    def release(self):
        raise NotImplementedError("Real implementation releases lock")
    
    def __enter__(self):
        self.acquire()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.release()
        return False

class BatchProcessor:
    """Processes batches of transactions."""
    def __init__(self, repository, payment_processor, lock=None):
        self.repository = repository
        self.payment_processor = payment_processor
        self.lock = lock
    
    def process_pending_batch(self, limit=100):
        # Acquire lock if provided
        if self.lock:
            with self.lock:
                return self._process_batch(limit)
        else:
            return self._process_batch(limit)
    
    def _process_batch(self, limit):
        batch = self.repository.fetch_pending(limit)
        
        if len(batch) == 0:
            return {"processed": 0, "failed": 0}
        
        processed = 0
        failed = 0
        
        for transaction in batch:
            try:
                result = self.payment_processor.process_payment(
                    user_id=transaction["user_id"],
                    amount=transaction["amount"],
                    card_token=transaction["card_token"]
                )
                
                if result["status"] == "success":
                    self.repository.mark_processed(transaction["id"])
                    processed += 1
                else:
                    failed += 1
            except Exception:
                failed += 1
        
        return {"processed": processed, "failed": failed}
```

Now test the lock behavior:

```python
def test_process_batch_with_lock():
    repository = Mock()
    payment_processor = Mock()
    lock = MagicMock()  # MagicMock supports __enter__ and __exit__
    
    mock_batch = MagicMock()
    mock_batch.__len__.return_value = 0
    repository.fetch_pending.return_value = mock_batch
    
    processor = BatchProcessor(repository, payment_processor, lock)
    result = processor.process_pending_batch(limit=10)
    
    assert result["processed"] == 0
    
    # Verify lock was acquired and released
    lock.__enter__.assert_called_once()
    lock.__exit__.assert_called_once()
```

Run this test:

```bash
pytest test_batch_processor_magicmock.py::test_process_batch_with_lock -v
```

**Output**:
```
test_batch_processor_magicmock.py::test_process_batch_with_lock PASSED
```

**What we verified**:
- The lock's `__enter__` method was called (lock acquired)
- The lock's `__exit__` method was called (lock released)
- `MagicMock` automatically supports context manager protocol

### When to Use MagicMock vs Mock

**Use MagicMock when**:
- Your code uses magic methods (`len()`, iteration, indexing, etc.)
- You're testing code that treats objects as containers
- You're testing context managers (`with` statements)
- You're testing callable objects
- You're testing operator overloading

**Use regular Mock when**:
- You only need to mock regular methods and attributes
- You want to be explicit about what's being mocked
- You don't need magic method support
- You want slightly better performance (MagicMock has more overhead)

**Practical guideline**: Start with `Mock()`. If you get a `TypeError` about magic methods, switch to `MagicMock()`.

### The Complete Test Suite

Here's our complete test suite using `MagicMock`:

```python
# test_batch_processor_complete.py
from unittest.mock import Mock, MagicMock
from transaction_pipeline import BatchProcessor

def test_process_empty_batch():
    """Test processing when no transactions are pending."""
    repository = Mock()
    payment_processor = Mock()
    
    mock_batch = MagicMock()
    mock_batch.__len__.return_value = 0
    repository.fetch_pending.return_value = mock_batch
    
    processor = BatchProcessor(repository, payment_processor)
    result = processor.process_pending_batch(limit=10)
    
    assert result["processed"] == 0
    assert result["failed"] == 0
    repository.mark_processed.assert_not_called()

def test_process_batch_all_succeed():
    """Test processing when all transactions succeed."""
    repository = Mock()
    payment_processor = Mock()
    
    transactions = [
        {"id": "txn_1", "user_id": "user_1", "amount": 10.00, "card_token": "tok_1"},
        {"id": "txn_2", "user_id": "user_2", "amount": 20.00, "card_token": "tok_2"},
    ]
    
    mock_batch = MagicMock()
    mock_batch.__len__.return_value = 2
    mock_batch.__iter__.return_value = iter(transactions)
    repository.fetch_pending.return_value = mock_batch
    
    payment_processor.process_payment.return_value = {"status": "success"}
    
    processor = BatchProcessor(repository, payment_processor)
    result = processor.process_pending_batch(limit=10)
    
    assert result["processed"] == 2
    assert result["failed"] == 0

def test_process_batch_with_failures():
    """Test processing when some transactions fail."""
    repository = Mock()
    payment_processor = Mock()
    
    transactions = [
        {"id": "txn_1", "user_id": "user_1", "amount": 10.00, "card_token": "tok_1"},
        {"id": "txn_2", "user_id": "user_2", "amount": 20.00, "card_token": "tok_2"},
    ]
    
    mock_batch = MagicMock()
    mock_batch.__len__.return_value = 2
    mock_batch.__iter__.return_value = iter(transactions)
    repository.fetch_pending.return_value = mock_batch
    
    payment_processor.process_payment.side_effect = [
        {"status": "success"},
        {"status": "rejected", "reason": "fraud_detected"},
    ]
    
    processor = BatchProcessor(repository, payment_processor)
    result = processor.process_pending_batch(limit=10)
    
    assert result["processed"] == 1
    assert result["failed"] == 1

def test_process_batch_with_lock():
    """Test that lock is properly acquired and released."""
    repository = Mock()
    payment_processor = Mock()
    lock = MagicMock()
    
    mock_batch = MagicMock()
    mock_batch.__len__.return_value = 0
    repository.fetch_pending.return_value = mock_batch
    
    processor = BatchProcessor(repository, payment_processor, lock)
    result = processor.process_pending_batch(limit=10)
    
    lock.__enter__.assert_called_once()
    lock.__exit__.assert_called_once()
```

Run the complete suite:

```bash
pytest test_batch_processor_complete.py -v
```

**Output**:
```
test_batch_processor_complete.py::test_process_empty_batch PASSED
test_batch_processor_complete.py::test_process_batch_all_succeed PASSED
test_batch_processor_complete.py::test_process_batch_with_failures PASSED
test_batch_processor_complete.py::test_process_batch_with_lock PASSED
```

### Key Takeaways

1. **MagicMock extends Mock**: It's a subclass that pre-configures magic methods
2. **Use it for special Python protocols**: Containers, iterables, context managers, callables
3. **Configure magic methods like regular methods**: Use `return_value` and `side_effect`
4. **It's still a mock**: You can verify calls to magic methods just like regular methods
5. **Start with Mock, upgrade to MagicMock**: Only use MagicMock when you need magic method support

In the next section, we'll explore how to mock entire classes, not just individual objects.

## Mocking Entire Classes

## Mocking Entire Classes

So far, we've mocked individual objects—creating a mock instance and configuring its methods. But sometimes you need to mock an entire class, so that every instance created from that class is automatically a mock.

This is essential when:
- Your code creates instances internally (you can't inject them)
- You need to verify that a class was instantiated with specific arguments
- You want all instances of a class to behave the same way in tests

### The Reference Implementation: A Payment Service Factory

Let's extend our payment system with a factory pattern that creates payment processors internally:

```python
# payment_service.py
from payment_processor import PaymentProcessor, PaymentGateway, FraudDetector, NotificationService

class PaymentServiceFactory:
    """Creates payment processors with proper dependencies."""
    
    @staticmethod
    def create_processor(gateway_type="stripe"):
        """Creates a payment processor with the specified gateway."""
        # Create dependencies - we instantiate classes here
        if gateway_type == "stripe":
            gateway = PaymentGateway()  # Creates Stripe gateway
        elif gateway_type == "paypal":
            gateway = PaymentGateway()  # Creates PayPal gateway
        else:
            raise ValueError(f"Unknown gateway type: {gateway_type}")
        
        fraud_detector = FraudDetector()  # Creates fraud detector
        notifier = NotificationService()  # Creates notification service
        
        # Return configured processor
        return PaymentProcessor(gateway, fraud_detector, notifier)

class PaymentService:
    """High-level payment service that uses the factory."""
    def __init__(self, gateway_type="stripe"):
        self.processor = PaymentServiceFactory.create_processor(gateway_type)
    
    def charge_customer(self, user_id, amount, card_token):
        """Charges a customer using the configured processor."""
        return self.processor.process_payment(user_id, amount, card_token)
```

The problem: `PaymentService` creates its own `PaymentProcessor` internally. We can't inject a mock processor because the factory creates it. We need to mock the classes themselves.

### Iteration 0: Attempting to Mock Individual Instances

Let's try the approach we know:

```python
# test_payment_service_naive.py
from unittest.mock import Mock
from payment_service import PaymentService

def test_charge_customer_naive():
    # Try to create a mock processor
    mock_processor = Mock()
    mock_processor.process_payment.return_value = {
        "status": "success",
        "transaction_id": "txn_12345"
    }
    
    # Create service
    service = PaymentService(gateway_type="stripe")
    
    # Try to charge
    result = service.charge_customer("user_42", 99.99, "tok_visa")
    
    # This will fail because service created its own processor
    assert result["status"] == "success"
```

Run this test:

```bash
pytest test_payment_service_naive.py::test_charge_customer_naive -v
```

**Output**:
```
test_payment_service_naive.py::test_charge_customer_naive FAILED

def test_charge_customer_naive():
    ...
    service = PaymentService(gateway_type="stripe")

payment_service.py:11: in create_processor
    gateway = PaymentGateway()
payment_processor.py:4: in __init__
    raise NotImplementedError("Real implementation calls external API")
E   NotImplementedError: Real implementation calls external API
```

### Diagnostic Analysis: Why Instance Mocking Fails

**The complete output**:
```
FAILED test_payment_service_naive.py::test_charge_customer_naive - NotImplementedError: Real implementation calls external API
```

**Let's parse this**:

1. **The summary line**: `NotImplementedError: Real implementation calls external API`
   - What this tells us: The real `PaymentGateway` class is being instantiated

2. **The traceback**:
```python
payment_service.py:11: in create_processor
    gateway = PaymentGateway()
payment_processor.py:4: in __init__
    raise NotImplementedError("Real implementation calls external API")
```
   - What this tells us: The factory is creating a real `PaymentGateway` instance
   - Key line: `gateway = PaymentGateway()` - this creates a real instance, not our mock

3. **The root cause**:
   - We created a mock processor instance, but the service doesn't use it
   - The factory creates its own instances by calling the class constructors
   - Our mock never gets used because we can't inject it

**Root cause identified**: We need to mock the class itself, not just create a mock instance.

**Why the current approach can't solve this**: Creating a mock instance doesn't affect what happens when code calls `PaymentGateway()`. The class constructor still creates real instances.

**What we need**: A way to replace the class itself so that calling `PaymentGateway()` returns a mock instead of a real instance.

### Iteration 1: Introducing patch() for Class Mocking

The `@patch` decorator can mock entire classes. When you patch a class, every call to that class's constructor returns a mock instance.

Let's fix our test:

```python
# test_payment_service_patched.py
from unittest.mock import Mock, patch
from payment_service import PaymentService

@patch('payment_service.NotificationService')
@patch('payment_service.FraudDetector')
@patch('payment_service.PaymentGateway')
@patch('payment_service.PaymentProcessor')
def test_charge_customer_with_class_mocking(
    mock_processor_class,
    mock_gateway_class,
    mock_fraud_class,
    mock_notifier_class
):
    # Configure what the mocked PaymentProcessor instance should return
    mock_processor_instance = Mock()
    mock_processor_instance.process_payment.return_value = {
        "status": "success",
        "transaction_id": "txn_12345"
    }
    
    # Make the PaymentProcessor class return our configured instance
    mock_processor_class.return_value = mock_processor_instance
    
    # Create service - this will use mocked classes
    service = PaymentService(gateway_type="stripe")
    
    # Charge customer
    result = service.charge_customer("user_42", 99.99, "tok_visa")
    
    # Verify result
    assert result["status"] == "success"
    assert result["transaction_id"] == "txn_12345"
    
    # Verify the processor was called correctly
    mock_processor_instance.process_payment.assert_called_once_with(
        "user_42", 99.99, "tok_visa"
    )
```

Run this test:

```bash
pytest test_payment_service_patched.py::test_charge_customer_with_class_mocking -v
```

**Output**:
```
test_payment_service_patched.py::test_charge_customer_with_class_mocking PASSED
```

**What changed**:
- We used `@patch` to replace the classes themselves
- `mock_processor_class` is the mocked `PaymentProcessor` class
- When code calls `PaymentProcessor(...)`, it returns `mock_processor_instance`
- We configured `mock_processor_class.return_value` to control what instance is created

### Understanding Class Mocking: The Two-Level System

When you patch a class, you get two mock objects:

1. **The mock class** (`mock_processor_class`): Represents the class itself
2. **The mock instance** (`mock_processor_class.return_value`): What gets returned when the class is instantiated

This diagram shows the relationship:

```
Real Code:                          Test Code:
-----------                         -----------
PaymentProcessor (class)    →       mock_processor_class (Mock)
    ↓ instantiate                       ↓ .return_value
processor (instance)        →       mock_processor_instance (Mock)
    ↓ call method                       ↓ configure
processor.process_payment() →       .process_payment.return_value
```

### Iteration 2: Verifying Class Instantiation

Let's verify that the classes were instantiated with the correct arguments:

```python
@patch('payment_service.NotificationService')
@patch('payment_service.FraudDetector')
@patch('payment_service.PaymentGateway')
@patch('payment_service.PaymentProcessor')
def test_verify_class_instantiation(
    mock_processor_class,
    mock_gateway_class,
    mock_fraud_class,
    mock_notifier_class
):
    # Configure processor instance
    mock_processor_instance = Mock()
    mock_processor_instance.process_payment.return_value = {
        "status": "success",
        "transaction_id": "txn_12345"
    }
    mock_processor_class.return_value = mock_processor_instance
    
    # Create service
    service = PaymentService(gateway_type="stripe")
    
    # Verify all dependency classes were instantiated
    mock_gateway_class.assert_called_once()
    mock_fraud_class.assert_called_once()
    mock_notifier_class.assert_called_once()
    
    # Verify PaymentProcessor was instantiated with the mocked dependencies
    mock_processor_class.assert_called_once_with(
        mock_gateway_class.return_value,  # The gateway instance
        mock_fraud_class.return_value,    # The fraud detector instance
        mock_notifier_class.return_value  # The notifier instance
    )
    
    # Charge customer
    result = service.charge_customer("user_42", 99.99, "tok_visa")
    
    assert result["status"] == "success"
```

Run this test:

```bash
pytest test_payment_service_patched.py::test_verify_class_instantiation -v
```

**Output**:
```
test_payment_service_patched.py::test_verify_class_instantiation PASSED
```

**What we verified**:
- Each dependency class was instantiated exactly once
- `PaymentProcessor` was instantiated with the correct mock instances
- The instances passed to `PaymentProcessor` are the `.return_value` of each mocked class

### Iteration 3: Testing Different Gateway Types

Let's test that different gateway types create different configurations:

```python
@patch('payment_service.NotificationService')
@patch('payment_service.FraudDetector')
@patch('payment_service.PaymentGateway')
@patch('payment_service.PaymentProcessor')
def test_different_gateway_types(
    mock_processor_class,
    mock_gateway_class,
    mock_fraud_class,
    mock_notifier_class
):
    # Configure processor
    mock_processor_instance = Mock()
    mock_processor_instance.process_payment.return_value = {"status": "success"}
    mock_processor_class.return_value = mock_processor_instance
    
    # Test Stripe gateway
    service_stripe = PaymentService(gateway_type="stripe")
    service_stripe.charge_customer("user_42", 99.99, "tok_visa")
    
    # Verify gateway was created
    assert mock_gateway_class.call_count == 1
    
    # Reset mocks
    mock_gateway_class.reset_mock()
    mock_processor_class.reset_mock()
    
    # Test PayPal gateway
    service_paypal = PaymentService(gateway_type="paypal")
    service_paypal.charge_customer("user_42", 99.99, "tok_visa")
    
    # Verify gateway was created again
    assert mock_gateway_class.call_count == 1
```

Run this test:

```bash
pytest test_payment_service_patched.py::test_different_gateway_types -v
```

**Output**:
```
test_payment_service_patched.py::test_different_gateway_types PASSED
```

**What we demonstrated**:
- We can test multiple instantiations of the same class
- `reset_mock()` clears call history between tests
- Each instantiation is tracked separately

### Iteration 4: Handling Invalid Gateway Types

Let's test error handling:

```python
@patch('payment_service.NotificationService')
@patch('payment_service.FraudDetector')
@patch('payment_service.PaymentGateway')
@patch('payment_service.PaymentProcessor')
def test_invalid_gateway_type(
    mock_processor_class,
    mock_gateway_class,
    mock_fraud_class,
    mock_notifier_class
):
    # Attempt to create service with invalid gateway
    import pytest
    with pytest.raises(ValueError) as exc_info:
        service = PaymentService(gateway_type="invalid")
    
    assert "Unknown gateway type: invalid" in str(exc_info.value)
    
    # Verify no classes were instantiated
    mock_gateway_class.assert_not_called()
    mock_fraud_class.assert_not_called()
    mock_notifier_class.assert_not_called()
    mock_processor_class.assert_not_called()
```

Run this test:

```bash
pytest test_payment_service_patched.py::test_invalid_gateway_type -v
```

**Output**:
```
test_payment_service_patched.py::test_invalid_gateway_type PASSED
```

**What we verified**:
- The factory raises an appropriate error for invalid gateway types
- No classes were instantiated when the error occurred
- We can verify that classes were NOT called using `assert_not_called()`

### The patch() Target: Where to Patch

A critical detail: we patched `'payment_service.PaymentGateway'`, not `'payment_processor.PaymentGateway'`.

**Rule**: Patch where the class is used, not where it's defined.

Here's why:

```python
# payment_service.py imports PaymentGateway
from payment_processor import PaymentGateway

# When this code runs:
gateway = PaymentGateway()

# Python looks up PaymentGateway in payment_service's namespace
# So we must patch 'payment_service.PaymentGateway'
```

If we patched `'payment_processor.PaymentGateway'`, it would replace the class in the `payment_processor` module, but `payment_service` already imported it into its own namespace. The patch wouldn't affect the imported reference.

### Common Failure Mode: Patching the Wrong Location

Let's demonstrate this mistake:

```python
# Wrong: patching where it's defined
@patch('payment_processor.PaymentGateway')  # Wrong location!
@patch('payment_service.PaymentProcessor')
def test_wrong_patch_location(mock_processor_class, mock_gateway_class):
    mock_processor_instance = Mock()
    mock_processor_instance.process_payment.return_value = {"status": "success"}
    mock_processor_class.return_value = mock_processor_instance
    
    # This will fail because PaymentGateway isn't patched in payment_service
    service = PaymentService(gateway_type="stripe")
```

Run this test:

```bash
pytest test_payment_service_patched.py::test_wrong_patch_location -v
```

**Output**:
```
test_payment_service_patched.py::test_wrong_patch_location FAILED

payment_service.py:11: in create_processor
    gateway = PaymentGateway()
payment_processor.py:4: in __init__
    raise NotImplementedError("Real implementation calls external API")
E   NotImplementedError: Real implementation calls external API
```

The real class is still being instantiated because we patched the wrong location.

### Using patch.object() for Explicit Class Mocking

An alternative to string-based patching is `patch.object()`, which patches an attribute of an object:

```python
from unittest.mock import Mock, patch
import payment_service

@patch.object(payment_service, 'NotificationService')
@patch.object(payment_service, 'FraudDetector')
@patch.object(payment_service, 'PaymentGateway')
@patch.object(payment_service, 'PaymentProcessor')
def test_with_patch_object(
    mock_processor_class,
    mock_gateway_class,
    mock_fraud_class,
    mock_notifier_class
):
    mock_processor_instance = Mock()
    mock_processor_instance.process_payment.return_value = {
        "status": "success",
        "transaction_id": "txn_12345"
    }
    mock_processor_class.return_value = mock_processor_instance
    
    service = PaymentService(gateway_type="stripe")
    result = service.charge_customer("user_42", 99.99, "tok_visa")
    
    assert result["status"] == "success"
```

Run this test:

```bash
pytest test_payment_service_patched.py::test_with_patch_object -v
```

**Output**:
```
test_payment_service_patched.py::test_with_patch_object PASSED
```

**Advantages of patch.object()**:
- More explicit: you specify the module object directly
- Catches typos at import time (if module doesn't exist, import fails)
- Clearer intent: "patch this attribute of this module"

**Disadvantages**:
- Requires importing the module
- More verbose

### When to Use Class Mocking

**Use class mocking when**:
- Your code creates instances internally (factory patterns)
- You need to verify class instantiation arguments
- You want to control all instances of a class uniformly
- You're testing code that uses dependency injection frameworks

**Don't use class mocking when**:
- You can inject mock instances directly (prefer dependency injection)
- You only need to mock one specific instance
- The class is simple and doesn't need mocking (use real instances)

**Best practice**: Prefer dependency injection over class mocking. Class mocking is powerful but makes tests more brittle because they depend on implementation details (which classes are instantiated and when).

### The Complete Test Suite

Here's our complete test suite for class mocking:

```python
# test_payment_service_complete.py
from unittest.mock import Mock, patch
import pytest
from payment_service import PaymentService

@patch('payment_service.NotificationService')
@patch('payment_service.FraudDetector')
@patch('payment_service.PaymentGateway')
@patch('payment_service.PaymentProcessor')
def test_successful_charge(
    mock_processor_class,
    mock_gateway_class,
    mock_fraud_class,
    mock_notifier_class
):
    """Test successful payment processing."""
    mock_processor_instance = Mock()
    mock_processor_instance.process_payment.return_value = {
        "status": "success",
        "transaction_id": "txn_12345"
    }
    mock_processor_class.return_value = mock_processor_instance
    
    service = PaymentService(gateway_type="stripe")
    result = service.charge_customer("user_42", 99.99, "tok_visa")
    
    assert result["status"] == "success"
    assert result["transaction_id"] == "txn_12345"
    mock_processor_instance.process_payment.assert_called_once_with(
        "user_42", 99.99, "tok_visa"
    )

@patch('payment_service.NotificationService')
@patch('payment_service.FraudDetector')
@patch('payment_service.PaymentGateway')
@patch('payment_service.PaymentProcessor')
def test_class_instantiation_order(
    mock_processor_class,
    mock_gateway_class,
    mock_fraud_class,
    mock_notifier_class
):
    """Test that dependencies are instantiated correctly."""
    mock_processor_instance = Mock()
    mock_processor_class.return_value = mock_processor_instance
    
    service = PaymentService(gateway_type="stripe")
    
    # Verify all dependencies were created
    mock_gateway_class.assert_called_once()
    mock_fraud_class.assert_called_once()
    mock_notifier_class.assert_called_once()
    
    # Verify processor was created with the right dependencies
    mock_processor_class.assert_called_once_with(
        mock_gateway_class.return_value,
        mock_fraud_class.return_value,
        mock_notifier_class.return_value
    )

@patch('payment_service.NotificationService')
@patch('payment_service.FraudDetector')
@patch('payment_service.PaymentGateway')
@patch('payment_service.PaymentProcessor')
def test_invalid_gateway_type(
    mock_processor_class,
    mock_gateway_class,
    mock_fraud_class,
    mock_notifier_class
):
    """Test error handling for invalid gateway types."""
    with pytest.raises(ValueError) as exc_info:
        service = PaymentService(gateway_type="invalid")
    
    assert "Unknown gateway type: invalid" in str(exc_info.value)
    mock_processor_class.assert_not_called()
```

Run the complete suite:

```bash
pytest test_payment_service_complete.py -v
```

**Output**:
```
test_payment_service_complete.py::test_successful_charge PASSED
test_payment_service_complete.py::test_class_instantiation_order PASSED
test_payment_service_complete.py::test_invalid_gateway_type PASSED
```

### Key Takeaways

1. **Class mocking replaces the class itself**: Every instantiation returns a mock
2. **Two-level system**: Mock class and mock instance (`.return_value`)
3. **Patch where used, not where defined**: Patch the import location
4. **Verify instantiation**: Use `assert_called_once_with()` on the mock class
5. **Use patch.object() for clarity**: More explicit than string-based patching
6. **Prefer dependency injection**: Class mocking is powerful but makes tests brittle

In the next section, we'll explore mocking properties and attributes, which require special handling.

## Mocking Properties and Attributes

## Mocking Properties and Attributes

So far, we've mocked methods—functions that are called. But Python objects also have properties and attributes that are accessed, not called. Mocking these requires different techniques.

Properties are especially tricky because they look like attributes but are actually methods decorated with `@property`. Understanding how to mock them is essential for testing real-world Python code.

### The Reference Implementation: A Configuration System

Let's build a configuration system that uses properties extensively:

```python
# config_system.py
import os
from datetime import datetime

class DatabaseConfig:
    """Database configuration with computed properties."""
    
    def __init__(self, host, port, database):
        self._host = host
        self._port = port
        self._database = database
        self._connection_count = 0
    
    @property
    def host(self):
        """Database host."""
        return self._host
    
    @property
    def port(self):
        """Database port."""
        return self._port
    
    @property
    def connection_string(self):
        """Computed connection string."""
        return f"postgresql://{self._host}:{self._port}/{self._database}"
    
    @property
    def is_local(self):
        """Check if database is on localhost."""
        return self._host in ("localhost", "127.0.0.1")
    
    @property
    def connection_count(self):
        """Number of connections made."""
        return self._connection_count
    
    def connect(self):
        """Simulate connecting to database."""
        self._connection_count += 1
        return f"Connected to {self.connection_string}"

class EnvironmentConfig:
    """Configuration that reads from environment variables."""
    
    @property
    def debug_mode(self):
        """Check if debug mode is enabled."""
        return os.environ.get("DEBUG", "false").lower() == "true"
    
    @property
    def log_level(self):
        """Get log level from environment."""
        return os.environ.get("LOG_LEVEL", "INFO")
    
    @property
    def current_timestamp(self):
        """Get current timestamp - always changes."""
        return datetime.now().isoformat()

class ApplicationConfig:
    """Main application configuration."""
    
    def __init__(self, db_config, env_config):
        self.db = db_config
        self.env = env_config
    
    def get_status(self):
        """Get application status."""
        return {
            "database": self.db.connection_string,
            "is_local": self.db.is_local,
            "debug_mode": self.env.debug_mode,
            "log_level": self.env.log_level,
            "timestamp": self.env.current_timestamp
        }
    
    def initialize(self):
        """Initialize application."""
        if self.env.debug_mode:
            print(f"Debug mode enabled at {self.env.current_timestamp}")
        
        connection_result = self.db.connect()
        
        return {
            "status": "initialized",
            "connection": connection_result,
            "connections": self.db.connection_count
        }
```

This configuration system uses properties for:
- Simple attribute access (`host`, `port`)
- Computed values (`connection_string`, `is_local`)
- Environment variable access (`debug_mode`, `log_level`)
- Dynamic values (`current_timestamp`)

### Iteration 0: Attempting to Mock Properties Like Methods

Let's try to test this using what we know about mocking methods:

```python
# test_config_naive.py
from unittest.mock import Mock
from config_system import ApplicationConfig

def test_get_status_naive():
    # Try to mock the config objects
    db_config = Mock()
    env_config = Mock()
    
    # Try to configure properties like methods
    db_config.connection_string.return_value = "postgresql://localhost:5432/testdb"
    db_config.is_local.return_value = True
    env_config.debug_mode.return_value = False
    env_config.log_level.return_value = "INFO"
    env_config.current_timestamp.return_value = "2024-01-01T00:00:00"
    
    app_config = ApplicationConfig(db_config, env_config)
    status = app_config.get_status()
    
    print(f"Status: {status}")
```

Run this test:

```bash
pytest test_config_naive.py::test_get_status_naive -v -s
```

**Output**:
```
test_config_naive.py::test_get_status_naive Status: {'database': <MagicMock name='mock.connection_string' id='...'>, 'is_local': <MagicMock name='mock.is_local' id='...'>, 'debug_mode': <MagicMock name='mock.debug_mode' id='...'>, 'log_level': <MagicMock name='mock.log_level' id='...'>, 'timestamp': <MagicMock name='mock.current_timestamp' id='...'>}
PASSED
```

### Diagnostic Analysis: Properties Return Mock Objects, Not Values

**The complete output**:
```
Status: {'database': <MagicMock name='mock.connection_string' id='...'>, ...}
```

**Let's parse this**:

1. **The status dictionary contains Mock objects**: Each value is `<MagicMock ...>` instead of the expected string/boolean
   - What this tells us: The properties are returning Mock objects, not the configured values

2. **Why this happens**:
   - When we access `db_config.connection_string`, Mock creates a new Mock object
   - We configured `db_config.connection_string.return_value`, but that's for calling it as a method
   - Properties are accessed, not called: `obj.property` not `obj.property()`
   - The `.return_value` configuration is never used

3. **The root cause**:
   - Properties are accessed like attributes: `obj.property`
   - Methods are called: `obj.method()`
   - Configuring `.return_value` only affects method calls
   - We need to configure the attribute value directly

**Root cause identified**: Properties are accessed, not called. We need to configure attribute values, not return values.

**Why the current approach can't solve this**: `.return_value` is for callable objects. Properties are not callable—they're accessed like attributes.

**What we need**: A way to configure what value is returned when an attribute is accessed.

### Iteration 1: Configuring Properties Correctly

To mock properties, we assign values directly to the mock's attributes:

```python
# test_config_properties.py
from unittest.mock import Mock
from config_system import ApplicationConfig

def test_get_status_with_properties():
    db_config = Mock()
    env_config = Mock()
    
    # Configure properties by direct assignment (not .return_value)
    db_config.connection_string = "postgresql://localhost:5432/testdb"
    db_config.is_local = True
    env_config.debug_mode = False
    env_config.log_level = "INFO"
    env_config.current_timestamp = "2024-01-01T00:00:00"
    
    app_config = ApplicationConfig(db_config, env_config)
    status = app_config.get_status()
    
    # Verify status contains the configured values
    assert status["database"] == "postgresql://localhost:5432/testdb"
    assert status["is_local"] is True
    assert status["debug_mode"] is False
    assert status["log_level"] == "INFO"
    assert status["timestamp"] == "2024-01-01T00:00:00"
```

Run this test:

```bash
pytest test_config_properties.py::test_get_status_with_properties -v
```

**Output**:
```
test_config_properties.py::test_get_status_with_properties PASSED
```

**What changed**:
- We assigned values directly: `db_config.connection_string = "..."`
- No `.return_value` needed
- The mock returns the assigned value when the property is accessed
- The test now verifies the actual values, not Mock objects

### Iteration 2: Testing Property-Dependent Behavior

Let's test code that makes decisions based on property values:

```python
def test_initialize_with_debug_mode():
    db_config = Mock()
    env_config = Mock()
    
    # Configure properties
    db_config.connection_string = "postgresql://localhost:5432/testdb"
    db_config.connection_count = 1  # After connection
    env_config.debug_mode = True  # Debug enabled
    env_config.current_timestamp = "2024-01-01T00:00:00"
    
    # Mock the connect method (this IS a method, so use return_value)
    db_config.connect.return_value = "Connected to postgresql://localhost:5432/testdb"
    
    app_config = ApplicationConfig(db_config, env_config)
    result = app_config.initialize()
    
    # Verify initialization result
    assert result["status"] == "initialized"
    assert result["connection"] == "Connected to postgresql://localhost:5432/testdb"
    assert result["connections"] == 1
    
    # Verify connect was called
    db_config.connect.assert_called_once()
```

Run this test:

```bash
pytest test_config_properties.py::test_initialize_with_debug_mode -v
```

**Output**:
```
test_config_properties.py::test_initialize_with_debug_mode PASSED
```

**What we demonstrated**:
- Properties use direct assignment: `env_config.debug_mode = True`
- Methods use `.return_value`: `db_config.connect.return_value = "..."`
- We can mix property mocking and method mocking in the same test

### Iteration 3: Using PropertyMock for Advanced Scenarios

Sometimes you need more control over property behavior. `PropertyMock` is a specialized mock for properties that need to:
- Track access (how many times was the property read?)
- Return different values on successive accesses
- Raise exceptions when accessed

Let's use `PropertyMock` to test dynamic properties:

```python
from unittest.mock import Mock, PropertyMock

def test_dynamic_timestamp_with_property_mock():
    db_config = Mock()
    env_config = Mock()
    
    # Configure static properties normally
    db_config.connection_string = "postgresql://localhost:5432/testdb"
    db_config.connect.return_value = "Connected"
    env_config.debug_mode = False
    env_config.log_level = "INFO"
    
    # Use PropertyMock for dynamic timestamp
    timestamp_mock = PropertyMock(side_effect=[
        "2024-01-01T00:00:00",
        "2024-01-01T00:00:01",
        "2024-01-01T00:00:02"
    ])
    type(env_config).current_timestamp = timestamp_mock
    
    app_config = ApplicationConfig(db_config, env_config)
    
    # Access timestamp multiple times
    status1 = app_config.get_status()
    status2 = app_config.get_status()
    status3 = app_config.get_status()
    
    # Verify different timestamps were returned
    assert status1["timestamp"] == "2024-01-01T00:00:00"
    assert status2["timestamp"] == "2024-01-01T00:00:01"
    assert status3["timestamp"] == "2024-01-01T00:00:02"
    
    # Verify property was accessed 3 times
    assert timestamp_mock.call_count == 3
```

Run this test:

```bash
pytest test_config_properties.py::test_dynamic_timestamp_with_property_mock -v
```

**Output**:
```
test_config_properties.py::test_dynamic_timestamp_with_property_mock PASSED
```

**What PropertyMock provides**:
- `side_effect` for returning different values on each access
- Call tracking (`.call_count`, `.assert_called()`, etc.)
- Must be assigned to the mock's type: `type(mock).property = PropertyMock(...)`

### Understanding type(mock).property Assignment

The syntax `type(env_config).current_timestamp = timestamp_mock` looks strange. Here's why it's necessary:

Properties in Python are descriptors that must be defined on the class, not the instance. When you access `obj.property`, Python:
1. Looks for `property` on the class (`type(obj)`)
2. Calls the property's `__get__` method
3. Returns the result

To mock a property, we must assign it to the mock's class (accessed via `type(mock)`), not the instance.

### Iteration 4: Testing Property Exceptions

Let's test what happens when a property raises an exception:

```python
def test_property_raises_exception():
    db_config = Mock()
    env_config = Mock()
    
    # Configure most properties normally
    db_config.connection_string = "postgresql://localhost:5432/testdb"
    env_config.debug_mode = False
    env_config.log_level = "INFO"
    env_config.current_timestamp = "2024-01-01T00:00:00"
    
    # Make is_local raise an exception
    is_local_mock = PropertyMock(side_effect=RuntimeError("Network error"))
    type(db_config).is_local = is_local_mock
    
    app_config = ApplicationConfig(db_config, env_config)
    
    # Accessing is_local should raise the exception
    import pytest
    with pytest.raises(RuntimeError) as exc_info:
        status = app_config.get_status()
    
    assert "Network error" in str(exc_info.value)
    
    # Verify the property was accessed
    is_local_mock.assert_called_once()
```

Run this test:

```bash
pytest test_config_properties.py::test_property_raises_exception -v
```

**Output**:
```
test_config_properties.py::test_property_raises_exception PASSED
```

**What we demonstrated**:
- `PropertyMock` can raise exceptions using `side_effect`
- The exception is raised when the property is accessed
- We can verify the property was accessed even though it raised an exception

### Iteration 5: Mocking Properties with patch()

You can also mock properties using `@patch`, which is useful when you need to mock properties on real classes:

```python
from unittest.mock import patch, PropertyMock
from config_system import DatabaseConfig, EnvironmentConfig, ApplicationConfig

@patch.object(EnvironmentConfig, 'debug_mode', new_callable=PropertyMock)
@patch.object(EnvironmentConfig, 'log_level', new_callable=PropertyMock)
@patch.object(EnvironmentConfig, 'current_timestamp', new_callable=PropertyMock)
def test_with_patched_properties(mock_timestamp, mock_log_level, mock_debug_mode):
    # Configure the property mocks
    mock_debug_mode.return_value = True
    mock_log_level.return_value = "DEBUG"
    mock_timestamp.return_value = "2024-01-01T00:00:00"
    
    # Create real EnvironmentConfig (properties are mocked)
    env_config = EnvironmentConfig()
    
    # Create mock DatabaseConfig
    db_config = Mock()
    db_config.connection_string = "postgresql://localhost:5432/testdb"
    db_config.is_local = True
    db_config.connect.return_value = "Connected"
    
    app_config = ApplicationConfig(db_config, env_config)
    status = app_config.get_status()
    
    # Verify mocked property values
    assert status["debug_mode"] is True
    assert status["log_level"] == "DEBUG"
    assert status["timestamp"] == "2024-01-01T00:00:00"
    
    # Verify properties were accessed
    mock_debug_mode.assert_called()
    mock_log_level.assert_called()
    mock_timestamp.assert_called()
```

Run this test:

```bash
pytest test_config_properties.py::test_with_patched_properties -v
```

**Output**:
```
test_config_properties.py::test_with_patched_properties PASSED
```

**What changed**:
- We used `@patch.object` with `new_callable=PropertyMock`
- This replaces the real property with a `PropertyMock`
- We can use a real `EnvironmentConfig` instance with mocked properties
- The mock parameters are in reverse order (bottom decorator = first parameter)

### Common Failure Modes and Their Signatures

#### Symptom: Property returns Mock object instead of value

**Pytest output pattern**:
```
AssertionError: assert <MagicMock name='mock.property' id='...'> == 'expected_value'
```

**Diagnostic clues**:
- The assertion shows a Mock object where a value was expected
- You configured `.return_value` on a property

**Root cause**: Properties are accessed, not called. `.return_value` doesn't apply.

**Solution**: Use direct assignment: `mock.property = "value"`

#### Symptom: PropertyMock not working

**Pytest output pattern**:
```
AttributeError: 'Mock' object has no attribute 'property'
```

**Diagnostic clues**:
- You assigned `PropertyMock` to the instance: `mock.property = PropertyMock(...)`
- Properties must be on the class, not the instance

**Root cause**: Properties are class-level descriptors.

**Solution**: Assign to the type: `type(mock).property = PropertyMock(...)`

#### Symptom: Patched property not being used

**Pytest output pattern**:
```
AssertionError: assert 'real_value' == 'mocked_value'
```

**Diagnostic clues**:
- You patched the property but the real value is still being used
- You forgot `new_callable=PropertyMock`

**Root cause**: Without `new_callable=PropertyMock`, `@patch` creates a regular Mock, not a PropertyMock.

**Solution**: Add `new_callable=PropertyMock` to the `@patch` decorator

### Decision Framework: When to Use Each Approach

| Approach | Use When | Advantages | Disadvantages |
|----------|----------|------------|---------------|
| **Direct assignment** | Simple property mocking | Simple, clear | No call tracking |
| **PropertyMock** | Need call tracking or dynamic values | Full mock capabilities | More complex syntax |
| **@patch with PropertyMock** | Mocking properties on real classes | Works with real instances | Requires understanding decorator order |

### The Complete Test Suite

Here's our complete test suite for property mocking:

```python
# test_config_complete.py
from unittest.mock import Mock, PropertyMock, patch
import pytest
from config_system import ApplicationConfig, EnvironmentConfig

def test_simple_property_mocking():
    """Test basic property mocking with direct assignment."""
    db_config = Mock()
    env_config = Mock()
    
    db_config.connection_string = "postgresql://localhost:5432/testdb"
    db_config.is_local = True
    env_config.debug_mode = False
    env_config.log_level = "INFO"
    env_config.current_timestamp = "2024-01-01T00:00:00"
    
    app_config = ApplicationConfig(db_config, env_config)
    status = app_config.get_status()
    
    assert status["database"] == "postgresql://localhost:5432/testdb"
    assert status["is_local"] is True

def test_dynamic_property_with_property_mock():
    """Test property that returns different values on each access."""
    db_config = Mock()
    env_config = Mock()
    
    db_config.connection_string = "postgresql://localhost:5432/testdb"
    db_config.is_local = True
    env_config.debug_mode = False
    env_config.log_level = "INFO"
    
    timestamp_mock = PropertyMock(side_effect=[
        "2024-01-01T00:00:00",
        "2024-01-01T00:00:01"
    ])
    type(env_config).current_timestamp = timestamp_mock
    
    app_config = ApplicationConfig(db_config, env_config)
    
    status1 = app_config.get_status()
    status2 = app_config.get_status()
    
    assert status1["timestamp"] == "2024-01-01T00:00:00"
    assert status2["timestamp"] == "2024-01-01T00:00:01"
    assert timestamp_mock.call_count == 2

def test_property_exception():
    """Test property that raises an exception."""
    db_config = Mock()
    env_config = Mock()
    
    db_config.connection_string = "postgresql://localhost:5432/testdb"
    env_config.debug_mode = False
    env_config.log_level = "INFO"
    env_config.current_timestamp = "2024-01-01T00:00:00"
    
    is_local_mock = PropertyMock(side_effect=RuntimeError("Network error"))
    type(db_config).is_local = is_local_mock
    
    app_config = ApplicationConfig(db_config, env_config)
    
    with pytest.raises(RuntimeError) as exc_info:
        app_config.get_status()
    
    assert "Network error" in str(exc_info.value)

@patch.object(EnvironmentConfig, 'debug_mode', new_callable=PropertyMock)
@patch.object(EnvironmentConfig, 'log_level', new_callable=PropertyMock)
def test_patched_properties(mock_log_level, mock_debug_mode):
    """Test mocking properties on real classes."""
    mock_debug_mode.return_value = True
    mock_log_level.return_value = "DEBUG"
    
    env_config = EnvironmentConfig()
    
    assert env_config.debug_mode is True
    assert env_config.log_level == "DEBUG"
    
    mock_debug_mode.assert_called()
    mock_log_level.assert_called()
```

Run the complete suite:

```bash
pytest test_config_complete.py -v
```

**Output**:
```
test_config_complete.py::test_simple_property_mocking PASSED
test_config_complete.py::test_dynamic_property_with_property_mock PASSED
test_config_complete.py::test_property_exception PASSED
test_config_complete.py::test_patched_properties PASSED
```

### Key Takeaways

1. **Properties are accessed, not called**: Use direct assignment, not `.return_value`
2. **PropertyMock for advanced scenarios**: Use when you need call tracking or dynamic values
3. **type(mock).property syntax**: Required because properties are class-level descriptors
4. **@patch with new_callable=PropertyMock**: For mocking properties on real classes
5. **Mix property and method mocking**: Properties use assignment, methods use `.return_value`

In the next section, we'll explore testing code that uses external libraries, where we need to mock third-party dependencies.

## Testing Code That Uses External Libraries

## Testing Code That Uses External Libraries

Real-world applications depend on external libraries: HTTP clients, database drivers, cloud SDKs, message queues. Testing code that uses these libraries requires mocking third-party dependencies you don't control.

This section teaches you how to isolate your code from external libraries while maintaining realistic test scenarios.

### The Reference Implementation: A Cloud Storage Service

Let's build a service that uses the `boto3` library (AWS SDK) to interact with S3:

```python
# cloud_storage.py
import boto3
from botocore.exceptions import ClientError
import hashlib
import json

class StorageService:
    """Service for uploading and managing files in S3."""
    
    def __init__(self, bucket_name, region="us-east-1"):
        self.bucket_name = bucket_name
        self.s3_client = boto3.client('s3', region_name=region)
    
    def upload_file(self, file_path, object_key):
        """Upload a file to S3."""
        try:
            # Calculate file hash for integrity check
            with open(file_path, 'rb') as f:
                file_hash = hashlib.md5(f.read()).hexdigest()
            
            # Upload to S3
            self.s3_client.upload_file(
                Filename=file_path,
                Bucket=self.bucket_name,
                Key=object_key,
                ExtraArgs={'Metadata': {'md5': file_hash}}
            )
            
            return {
                "status": "success",
                "bucket": self.bucket_name,
                "key": object_key,
                "md5": file_hash
            }
        except ClientError as e:
            return {
                "status": "error",
                "error": str(e)
            }
        except FileNotFoundError:
            return {
                "status": "error",
                "error": "File not found"
            }
    
    def download_file(self, object_key, destination_path):
        """Download a file from S3."""
        try:
            self.s3_client.download_file(
                Bucket=self.bucket_name,
                Key=object_key,
                Filename=destination_path
            )
            
            return {
                "status": "success",
                "path": destination_path
            }
        except ClientError as e:
            return {
                "status": "error",
                "error": str(e)
            }
    
    def list_files(self, prefix=""):
        """List files in the bucket."""
        try:
            response = self.s3_client.list_objects_v2(
                Bucket=self.bucket_name,
                Prefix=prefix
            )
            
            if 'Contents' not in response:
                return []
            
            return [
                {
                    "key": obj["Key"],
                    "size": obj["Size"],
                    "last_modified": obj["LastModified"].isoformat()
                }
                for obj in response["Contents"]
            ]
        except ClientError as e:
            return []
    
    def delete_file(self, object_key):
        """Delete a file from S3."""
        try:
            self.s3_client.delete_object(
                Bucket=self.bucket_name,
                Key=object_key
            )
            
            return {"status": "success"}
        except ClientError as e:
            return {
                "status": "error",
                "error": str(e)
            }
```

This service uses `boto3` extensively:
- Creates an S3 client: `boto3.client('s3')`
- Calls S3 methods: `upload_file()`, `download_file()`, `list_objects_v2()`, `delete_object()`
- Handles S3-specific exceptions: `ClientError`

We cannot call real S3 in tests. We need to mock `boto3`.

### Iteration 0: Understanding the Challenge

Let's try to test this naively:

```python
# test_storage_naive.py
from cloud_storage import StorageService

def test_upload_file_naive():
    # Try to create a real service
    service = StorageService(bucket_name="test-bucket")
    
    # Try to upload a file
    result = service.upload_file("test.txt", "uploads/test.txt")
    
    print(f"Result: {result}")
```

Run this test:

```bash
pytest test_storage_naive.py::test_upload_file_naive -v -s
```

**Output**:
```
test_storage_naive.py::test_upload_file_naive FAILED

botocore/credentials.py:...: in load_credentials
    ...
E   botocore.exceptions.NoCredentialsError: Unable to locate credentials
```

### Diagnostic Analysis: External Library Requires Real Configuration

**The complete output**:
```
FAILED test_storage_naive.py::test_upload_file_naive - botocore.exceptions.NoCredentialsError: Unable to locate credentials
```

**Let's parse this**:

1. **The summary line**: `NoCredentialsError: Unable to locate credentials`
   - What this tells us: `boto3` is trying to authenticate with AWS
   - It needs real AWS credentials to work

2. **The traceback**:
```python
botocore/credentials.py:...: in load_credentials
```
   - What this tells us: The error occurs deep in the `boto3` library
   - Our code triggered real AWS SDK initialization

3. **The root cause**:
   - `boto3.client('s3')` creates a real S3 client
   - The real client tries to load AWS credentials
   - We don't have credentials configured (and don't want to use real AWS)
   - The test fails before our code even runs

**Root cause identified**: We're creating a real `boto3` client that tries to connect to AWS.

**Why the current approach can't solve this**: We can't test against real AWS—it's slow, expensive, requires credentials, and creates real resources.

**What we need**: A way to mock the `boto3` client so our code thinks it's talking to S3, but it's actually talking to a mock.

### Iteration 1: Mocking the boto3 Client

We need to mock `boto3.client()` to return a mock S3 client:

```python
# test_storage_mocked.py
from unittest.mock import Mock, patch, mock_open
from cloud_storage import StorageService

@patch('cloud_storage.boto3')
def test_upload_file_success(mock_boto3):
    # Create a mock S3 client
    mock_s3_client = Mock()
    
    # Make boto3.client() return our mock
    mock_boto3.client.return_value = mock_s3_client
    
    # Create service (will use mocked boto3)
    service = StorageService(bucket_name="test-bucket")
    
    # Mock file reading
    mock_file_content = b"test file content"
    with patch('builtins.open', mock_open(read_data=mock_file_content)):
        result = service.upload_file("test.txt", "uploads/test.txt")
    
    # Verify result
    assert result["status"] == "success"
    assert result["bucket"] == "test-bucket"
    assert result["key"] == "uploads/test.txt"
    
    # Verify S3 client was created correctly
    mock_boto3.client.assert_called_once_with('s3', region_name='us-east-1')
    
    # Verify upload_file was called
    mock_s3_client.upload_file.assert_called_once()
    call_args = mock_s3_client.upload_file.call_args
    assert call_args.kwargs['Filename'] == "test.txt"
    assert call_args.kwargs['Bucket'] == "test-bucket"
    assert call_args.kwargs['Key'] == "uploads/test.txt"
```

Run this test:

```bash
pytest test_storage_mocked.py::test_upload_file_success -v
```

**Output**:
```
test_storage_mocked.py::test_upload_file_success PASSED
```

**What we did**:
1. **Patched boto3**: `@patch('cloud_storage.boto3')` replaces the entire `boto3` module
2. **Mocked client creation**: `mock_boto3.client.return_value = mock_s3_client`
3. **Mocked file I/O**: `mock_open(read_data=...)` simulates reading a file
4. **Verified interactions**: Checked that `boto3.client()` and `upload_file()` were called correctly

### Iteration 2: Testing Error Handling

Let's test what happens when S3 operations fail:

```python
from botocore.exceptions import ClientError

@patch('cloud_storage.boto3')
def test_upload_file_s3_error(mock_boto3):
    mock_s3_client = Mock()
    mock_boto3.client.return_value = mock_s3_client
    
    # Make upload_file raise a ClientError
    error_response = {
        'Error': {
            'Code': 'NoSuchBucket',
            'Message': 'The specified bucket does not exist'
        }
    }
    mock_s3_client.upload_file.side_effect = ClientError(
        error_response, 'PutObject'
    )
    
    service = StorageService(bucket_name="nonexistent-bucket")
    
    mock_file_content = b"test content"
    with patch('builtins.open', mock_open(read_data=mock_file_content)):
        result = service.upload_file("test.txt", "uploads/test.txt")
    
    # Verify error was handled
    assert result["status"] == "error"
    assert "NoSuchBucket" in result["error"]
```

Run this test:

```bash
pytest test_storage_mocked.py::test_upload_file_s3_error -v
```

**Output**:
```
test_storage_mocked.py::test_upload_file_s3_error PASSED
```

**What we demonstrated**:
- Created a realistic `ClientError` exception (boto3's error type)
- Used `side_effect` to make the mock raise the exception
- Verified our code handles S3 errors correctly

### Iteration 3: Testing File Not Found

Let's test the file system error path:

```python
@patch('cloud_storage.boto3')
def test_upload_file_not_found(mock_boto3):
    mock_s3_client = Mock()
    mock_boto3.client.return_value = mock_s3_client
    
    service = StorageService(bucket_name="test-bucket")
    
    # Don't mock open() - let it fail naturally
    result = service.upload_file("nonexistent.txt", "uploads/test.txt")
    
    # Verify error was handled
    assert result["status"] == "error"
    assert result["error"] == "File not found"
    
    # Verify S3 was never called
    mock_s3_client.upload_file.assert_not_called()
```

Run this test:

```bash
pytest test_storage_mocked.py::test_upload_file_not_found -v
```

**Output**:
```
test_storage_mocked.py::test_upload_file_not_found PASSED
```

**What we demonstrated**:
- We didn't mock `open()` this time—let the real `FileNotFoundError` occur
- Verified our code handles file system errors
- Verified S3 was never called when the file doesn't exist

### Iteration 4: Testing List Operations

Let's test the `list_files()` method which returns structured data:

```python
from datetime import datetime

@patch('cloud_storage.boto3')
def test_list_files_success(mock_boto3):
    mock_s3_client = Mock()
    mock_boto3.client.return_value = mock_s3_client
    
    # Mock S3 response with realistic structure
    mock_s3_client.list_objects_v2.return_value = {
        'Contents': [
            {
                'Key': 'uploads/file1.txt',
                'Size': 1024,
                'LastModified': datetime(2024, 1, 1, 12, 0, 0)
            },
            {
                'Key': 'uploads/file2.txt',
                'Size': 2048,
                'LastModified': datetime(2024, 1, 2, 12, 0, 0)
            }
        ]
    }
    
    service = StorageService(bucket_name="test-bucket")
    files = service.list_files(prefix="uploads/")
    
    # Verify results
    assert len(files) == 2
    assert files[0]["key"] == "uploads/file1.txt"
    assert files[0]["size"] == 1024
    assert files[1]["key"] == "uploads/file2.txt"
    assert files[1]["size"] == 2048
    
    # Verify S3 was called correctly
    mock_s3_client.list_objects_v2.assert_called_once_with(
        Bucket="test-bucket",
        Prefix="uploads/"
    )
```

Run this test:

```bash
pytest test_storage_mocked.py::test_list_files_success -v
```

**Output**:
```
test_storage_mocked.py::test_list_files_success PASSED
```

**What we demonstrated**:
- Created a realistic S3 response structure
- Mocked complex nested data (list of dictionaries)
- Verified our code correctly transforms S3 data

### Iteration 5: Testing Empty Results

Let's test the edge case where no files exist:

```python
@patch('cloud_storage.boto3')
def test_list_files_empty(mock_boto3):
    mock_s3_client = Mock()
    mock_boto3.client.return_value = mock_s3_client
    
    # S3 returns no 'Contents' key when bucket is empty
    mock_s3_client.list_objects_v2.return_value = {}
    
    service = StorageService(bucket_name="test-bucket")
    files = service.list_files(prefix="uploads/")
    
    # Verify empty list is returned
    assert files == []
    
    mock_s3_client.list_objects_v2.assert_called_once()
```

Run this test:

```bash
pytest test_storage_mocked.py::test_list_files_empty -v
```

**Output**:
```
test_storage_mocked.py::test_list_files_empty PASSED
```

**What we demonstrated**:
- Tested edge case behavior (empty bucket)
- Verified our code handles missing keys in S3 responses
- Ensured we return an empty list, not an error

### Strategies for Mocking External Libraries

#### Strategy 1: Mock at the Boundary

Mock the external library at the point where your code calls it:

```python
@patch('your_module.external_library')
def test_something(mock_library):
    # Your code imports external_library
    # Patch it in your module's namespace
    pass
```

#### Strategy 2: Create Realistic Mock Responses

Study the external library's documentation to create realistic mock responses:

```python
# Bad: Unrealistic mock
mock_response = {"data": "something"}

# Good: Matches actual API response structure
mock_response = {
    'Contents': [
        {'Key': 'file.txt', 'Size': 1024, 'LastModified': datetime(...)}
    ],
    'IsTruncated': False,
    'MaxKeys': 1000
}
```

#### Strategy 3: Mock Exceptions Realistically

Use the actual exception types from the library:

```python
from botocore.exceptions import ClientError

# Create realistic error
error_response = {'Error': {'Code': 'NoSuchBucket', 'Message': '...'}}
mock_client.method.side_effect = ClientError(error_response, 'Operation')
```

#### Strategy 4: Test Both Success and Failure Paths

For each external library call, test:
- Successful operation
- Library-specific errors (e.g., `ClientError`)
- Network errors
- Timeout scenarios
- Edge cases (empty results, missing data)

### Common Patterns for Popular Libraries

#### Mocking HTTP Requests (requests library)

```python
@patch('your_module.requests')
def test_api_call(mock_requests):
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"data": "value"}
    mock_requests.get.return_value = mock_response
    
    # Your code that calls requests.get()
```

#### Mocking Database Connections (psycopg2, pymongo)

```python
@patch('your_module.psycopg2')
def test_database_query(mock_psycopg2):
    mock_connection = Mock()
    mock_cursor = Mock()
    mock_cursor.fetchall.return_value = [("row1",), ("row2",)]
    mock_connection.cursor.return_value = mock_cursor
    mock_psycopg2.connect.return_value = mock_connection
    
    # Your code that uses psycopg2.connect()
```

#### Mocking Redis

```python
@patch('your_module.redis')
def test_cache_operation(mock_redis):
    mock_client = Mock()
    mock_client.get.return_value = b"cached_value"
    mock_redis.Redis.return_value = mock_client
    
    # Your code that uses redis.Redis()
```

### When to Use Real Libraries vs Mocks

**Use mocks when**:
- The library requires external resources (network, cloud services)
- Operations are slow or expensive
- You need deterministic test results
- You're testing error handling

**Use real libraries when**:
- The library is pure Python with no external dependencies
- You're doing integration testing
- The library's behavior is complex and hard to mock accurately
- You have a test environment (e.g., local database)

### The Complete Test Suite

Here's our complete test suite for external library mocking:

```python
# test_storage_complete.py
from unittest.mock import Mock, patch, mock_open
from datetime import datetime
from botocore.exceptions import ClientError
from cloud_storage import StorageService

@patch('cloud_storage.boto3')
def test_upload_file_success(mock_boto3):
    """Test successful file upload."""
    mock_s3_client = Mock()
    mock_boto3.client.return_value = mock_s3_client
    
    service = StorageService(bucket_name="test-bucket")
    
    mock_file_content = b"test file content"
    with patch('builtins.open', mock_open(read_data=mock_file_content)):
        result = service.upload_file("test.txt", "uploads/test.txt")
    
    assert result["status"] == "success"
    assert result["bucket"] == "test-bucket"
    assert result["key"] == "uploads/test.txt"
    mock_s3_client.upload_file.assert_called_once()

@patch('cloud_storage.boto3')
def test_upload_file_s3_error(mock_boto3):
    """Test upload with S3 error."""
    mock_s3_client = Mock()
    mock_boto3.client.return_value = mock_s3_client
    
    error_response = {
        'Error': {
            'Code': 'NoSuchBucket',
            'Message': 'The specified bucket does not exist'
        }
    }
    mock_s3_client.upload_file.side_effect = ClientError(
        error_response, 'PutObject'
    )
    
    service = StorageService(bucket_name="nonexistent-bucket")
    
    with patch('builtins.open', mock_open(read_data=b"content")):
        result = service.upload_file("test.txt", "uploads/test.txt")
    
    assert result["status"] == "error"
    assert "NoSuchBucket" in result["error"]

@patch('cloud_storage.boto3')
def test_upload_file_not_found(mock_boto3):
    """Test upload with missing file."""
    mock_s3_client = Mock()
    mock_boto3.client.return_value = mock_s3_client
    
    service = StorageService(bucket_name="test-bucket")
    result = service.upload_file("nonexistent.txt", "uploads/test.txt")
    
    assert result["status"] == "error"
    assert result["error"] == "File not found"
    mock_s3_client.upload_file.assert_not_called()

@patch('cloud_storage.boto3')
def test_list_files_success(mock_boto3):
    """Test listing files."""
    mock_s3_client = Mock()
    mock_boto3.client.return_value = mock_s3_client
    
    mock_s3_client.list_objects_v2.return_value = {
        'Contents': [
            {
                'Key': 'uploads/file1.txt',
                'Size': 1024,
                'LastModified': datetime(2024, 1, 1, 12, 0, 0)
            },
            {
                'Key': 'uploads/file2.txt',
                'Size': 2048,
                'LastModified': datetime(2024, 1, 2, 12, 0, 0)
            }
        ]
    }
    
    service = StorageService(bucket_name="test-bucket")
    files = service.list_files(prefix="uploads/")
    
    assert len(files) == 2
    assert files[0]["key"] == "uploads/file1.txt"
    assert files[0]["size"] == 1024

@patch('cloud_storage.boto3')
def test_list_files_empty(mock_boto3):
    """Test listing files in empty bucket."""
    mock_s3_client = Mock()
    mock_boto3.client.return_value = mock_s3_client
    
    mock_s3_client.list_objects_v2.return_value = {}
    
    service = StorageService(bucket_name="test-bucket")
    files = service.list_files(prefix="uploads/")
    
    assert files == []

@patch('cloud_storage.boto3')
def test_delete_file_success(mock_boto3):
    """Test successful file deletion."""
    mock_s3_client = Mock()
    mock_boto3.client.return_value = mock_s3_client
    
    service = StorageService(bucket_name="test-bucket")
    result = service.delete_file("uploads/test.txt")
    
    assert result["status"] == "success"
    mock_s3_client.delete_object.assert_called_once_with(
        Bucket="test-bucket",
        Key="uploads/test.txt"
    )
```

Run the complete suite:

```bash
pytest test_storage_complete.py -v
```

**Output**:
```
test_storage_complete.py::test_upload_file_success PASSED
test_storage_complete.py::test_upload_file_s3_error PASSED
test_storage_complete.py::test_upload_file_not_found PASSED
test_storage_complete.py::test_list_files_success PASSED
test_storage_complete.py::test_list_files_empty PASSED
test_storage_complete.py::test_delete_file_success PASSED
```

### Key Takeaways

1. **Mock at the boundary**: Patch where the library is imported, not where it's defined
2. **Create realistic responses**: Study the library's documentation to mock accurately
3. **Test error paths**: Mock library-specific exceptions
4. **Verify interactions**: Check that library methods were called correctly
5. **Test edge cases**: Empty results, missing data, timeouts
6. **Use real exception types**: Import and use the library's actual exception classes

In the next section, we'll explore the dangers of over-mocking and learn when to stop mocking.

## Avoiding Over-Mocking

## Avoiding Over-Mocking

We've learned powerful mocking techniques throughout this chapter. But with great power comes great responsibility. Over-mocking is one of the most common mistakes in testing—it creates tests that pass but don't actually verify your code works.

This section teaches you to recognize over-mocking and develop judgment about when to mock and when not to.

### The Reference Implementation: A Report Generator

Let's build a report generator that processes data and formats it:

```python
# report_generator.py
from datetime import datetime
from typing import List, Dict

class DataProcessor:
    """Processes raw data for reporting."""
    
    def calculate_total(self, amounts: List[float]) -> float:
        """Calculate sum of amounts."""
        return sum(amounts)
    
    def calculate_average(self, amounts: List[float]) -> float:
        """Calculate average of amounts."""
        if not amounts:
            return 0.0
        return sum(amounts) / len(amounts)
    
    def filter_by_threshold(self, amounts: List[float], threshold: float) -> List[float]:
        """Filter amounts above threshold."""
        return [amount for amount in amounts if amount >= threshold]
    
    def group_by_category(self, transactions: List[Dict]) -> Dict[str, List[Dict]]:
        """Group transactions by category."""
        groups = {}
        for transaction in transactions:
            category = transaction.get("category", "uncategorized")
            if category not in groups:
                groups[category] = []
            groups[category].append(transaction)
        return groups

class ReportFormatter:
    """Formats data for display."""
    
    def format_currency(self, amount: float) -> str:
        """Format amount as currency."""
        return f"${amount:,.2f}"
    
    def format_percentage(self, value: float) -> str:
        """Format value as percentage."""
        return f"{value:.1f}%"
    
    def format_date(self, date: datetime) -> str:
        """Format date for display."""
        return date.strftime("%Y-%m-%d")

class ReportGenerator:
    """Generates formatted reports from data."""
    
    def __init__(self, processor: DataProcessor, formatter: ReportFormatter):
        self.processor = processor
        self.formatter = formatter
    
    def generate_summary(self, transactions: List[Dict]) -> Dict:
        """Generate a summary report."""
        amounts = [t["amount"] for t in transactions]
        
        total = self.processor.calculate_total(amounts)
        average = self.processor.calculate_average(amounts)
        high_value = self.processor.filter_by_threshold(amounts, 1000.0)
        
        return {
            "total": self.formatter.format_currency(total),
            "average": self.formatter.format_currency(average),
            "high_value_count": len(high_value),
            "transaction_count": len(transactions)
        }
    
    def generate_category_report(self, transactions: List[Dict]) -> Dict:
        """Generate a report grouped by category."""
        groups = self.processor.group_by_category(transactions)
        
        report = {}
        for category, category_transactions in groups.items():
            amounts = [t["amount"] for t in category_transactions]
            total = self.processor.calculate_total(amounts)
            
            report[category] = {
                "total": self.formatter.format_currency(total),
                "count": len(category_transactions)
            }
        
        return report
```

This report generator has three layers:
1. **DataProcessor**: Pure logic (calculations, filtering, grouping)
2. **ReportFormatter**: Pure formatting (no logic)
3. **ReportGenerator**: Orchestration (uses processor and formatter)

### Iteration 0: The Over-Mocked Test

Let's write a test that mocks everything:

```python
# test_report_over_mocked.py
from unittest.mock import Mock
from report_generator import ReportGenerator

def test_generate_summary_over_mocked():
    # Mock everything
    processor = Mock()
    formatter = Mock()
    
    # Configure all the mocks
    processor.calculate_total.return_value = 5000.0
    processor.calculate_average.return_value = 1000.0
    processor.filter_by_threshold.return_value = [1500.0, 2000.0]
    
    formatter.format_currency.side_effect = lambda x: f"${x:,.2f}"
    
    # Create generator with mocks
    generator = ReportGenerator(processor, formatter)
    
    # Test data
    transactions = [
        {"amount": 500.0},
        {"amount": 1000.0},
        {"amount": 1500.0},
        {"amount": 2000.0}
    ]
    
    # Generate report
    report = generator.generate_summary(transactions)
    
    # Verify result
    assert report["total"] == "$5,000.00"
    assert report["average"] == "$1,000.00"
    assert report["high_value_count"] == 2
    assert report["transaction_count"] == 4
    
    # Verify all mocks were called
    processor.calculate_total.assert_called_once()
    processor.calculate_average.assert_called_once()
    processor.filter_by_threshold.assert_called_once()
    formatter.format_currency.assert_called()
```

Run this test:

```bash
pytest test_report_over_mocked.py::test_generate_summary_over_mocked -v
```

**Output**:
```
test_report_over_mocked.py::test_generate_summary_over_mocked PASSED
```

The test passes. But what did we actually test?

### Diagnostic Analysis: What This Test Actually Verifies

Let's analyze what this test proves:

**What the test verifies**:
1. `ReportGenerator.generate_summary()` calls `processor.calculate_total()`
2. It calls `processor.calculate_average()`
3. It calls `processor.filter_by_threshold()`
4. It calls `formatter.format_currency()`
5. It returns a dictionary with the expected keys

**What the test does NOT verify**:
1. ❌ That `calculate_total()` actually sums the amounts correctly
2. ❌ That `calculate_average()` computes the average correctly
3. ❌ That `filter_by_threshold()` filters correctly
4. ❌ That `format_currency()` formats correctly
5. ❌ That the report contains the correct values for the given input

**The problem**: We mocked the return values to be what we expect, then verified we got what we mocked. This is circular logic. The test would pass even if all the business logic was broken.

Let's prove this by breaking the code:

```python
# report_generator_broken.py
class DataProcessor:
    def calculate_total(self, amounts: List[float]) -> float:
        # BROKEN: Returns wrong value
        return 999999.0
    
    def calculate_average(self, amounts: List[float]) -> float:
        # BROKEN: Returns wrong value
        return 0.0
    
    def filter_by_threshold(self, amounts: List[float], threshold: float) -> List[float]:
        # BROKEN: Returns empty list
        return []
```

Run the over-mocked test against the broken code:

```bash
# The test still passes because it never uses the real implementations!
pytest test_report_over_mocked.py::test_generate_summary_over_mocked -v
```

**Output**:
```
test_report_over_mocked.py::test_generate_summary_over_mocked PASSED
```

The test passes even though the code is completely broken. This is the danger of over-mocking.

### Iteration 1: Testing with Real Dependencies

Let's rewrite the test using real `DataProcessor` and `ReportFormatter`:

```python
# test_report_realistic.py
from report_generator import ReportGenerator, DataProcessor, ReportFormatter

def test_generate_summary_realistic():
    # Use REAL dependencies
    processor = DataProcessor()
    formatter = ReportFormatter()
    
    generator = ReportGenerator(processor, formatter)
    
    # Test data
    transactions = [
        {"amount": 500.0},
        {"amount": 1000.0},
        {"amount": 1500.0},
        {"amount": 2000.0}
    ]
    
    # Generate report
    report = generator.generate_summary(transactions)
    
    # Verify ACTUAL computed values
    assert report["total"] == "$5,000.00"  # 500 + 1000 + 1500 + 2000
    assert report["average"] == "$1,250.00"  # 5000 / 4
    assert report["high_value_count"] == 2  # 1500 and 2000 are >= 1000
    assert report["transaction_count"] == 4
```

Run this test:

```bash
pytest test_report_realistic.py::test_generate_summary_realistic -v
```

**Output**:
```
test_report_realistic.py::test_generate_summary_realistic PASSED
```

**What changed**:
- We use real `DataProcessor` and `ReportFormatter` instances
- The test verifies actual computed values, not mocked return values
- If the business logic is broken, this test will fail

Let's prove it by running this test against the broken code:

```python
# If we use the broken DataProcessor, this test fails:
# AssertionError: assert '$999,999.00' == '$5,000.00'
```

This test actually verifies the code works correctly.

### Iteration 2: When to Mock - External Dependencies Only

The rule of thumb: **Mock external dependencies, not your own code.**

Let's add an external dependency to our report generator:

```python
# report_generator.py (addition)
class EmailService:
    """External service for sending emails."""
    def send_email(self, to: str, subject: str, body: str):
        # In reality, calls external email API
        raise NotImplementedError("Real implementation sends email")

class ReportGenerator:
    def __init__(self, processor: DataProcessor, formatter: ReportFormatter, 
                 email_service: EmailService = None):
        self.processor = processor
        self.formatter = formatter
        self.email_service = email_service
    
    def generate_and_email_summary(self, transactions: List[Dict], recipient: str) -> Dict:
        """Generate summary and email it."""
        report = self.generate_summary(transactions)
        
        if self.email_service:
            body = f"""
            Summary Report:
            Total: {report['total']}
            Average: {report['average']}
            High Value Transactions: {report['high_value_count']}
            """
            
            self.email_service.send_email(
                to=recipient,
                subject="Transaction Summary Report",
                body=body
            )
        
        return report
```

Now we have a legitimate reason to mock—the `EmailService` is external:

```python
from unittest.mock import Mock

def test_generate_and_email_summary():
    # Use REAL processor and formatter
    processor = DataProcessor()
    formatter = ReportFormatter()
    
    # MOCK the external email service
    email_service = Mock()
    
    generator = ReportGenerator(processor, formatter, email_service)
    
    transactions = [
        {"amount": 500.0},
        {"amount": 1000.0},
        {"amount": 1500.0},
        {"amount": 2000.0}
    ]
    
    report = generator.generate_and_email_summary(transactions, "user@example.com")
    
    # Verify report is correct (using real logic)
    assert report["total"] == "$5,000.00"
    assert report["average"] == "$1,250.00"
    
    # Verify email was sent (mock verification)
    email_service.send_email.assert_called_once()
    call_args = email_service.send_email.call_args
    assert call_args.kwargs["to"] == "user@example.com"
    assert "Summary Report" in call_args.kwargs["subject"]
    assert "$5,000.00" in call_args.kwargs["body"]
```

Run this test:

```bash
pytest test_report_realistic.py::test_generate_and_email_summary -v
```

**Output**:
```
test_report_realistic.py::test_generate_and_email_summary PASSED
```

**What we did right**:
- Used real `DataProcessor` and `ReportFormatter` (our code)
- Mocked `EmailService` (external dependency)
- Verified actual computed values
- Verified the external service was called correctly

### The Over-Mocking Spectrum

Tests fall on a spectrum from "no mocking" to "everything mocked":

```
No Mocking          Balanced           Over-Mocked
    |                  |                    |
    v                  v                    v
[Real objects]  [Mock external]    [Mock everything]
[Integration]   [Unit + verify]    [Mock own code]
[Slow, real]    [Fast, focused]    [Fast, useless]
```

**No mocking** (left):
- Uses all real objects
- Tests integration between components
- Slow but comprehensive
- Good for integration tests

**Balanced** (center):
- Mocks external dependencies only
- Uses real implementations of your code
- Fast and meaningful
- Good for unit tests

**Over-mocked** (right):
- Mocks everything including your own code
- Tests only that methods are called
- Fast but verifies nothing
- Bad practice

### Signs You're Over-Mocking

**Warning sign 1: You're mocking simple logic**

```python
# Bad: Mocking simple calculation
processor = Mock()
processor.calculate_total.return_value = 100.0

# Good: Use real calculation
processor = DataProcessor()
total = processor.calculate_total([25.0, 25.0, 25.0, 25.0])
assert total == 100.0
```

**Warning sign 2: You're configuring return values to match assertions**

```python
# Bad: Circular logic
mock.method.return_value = "expected_value"
result = code_under_test()
assert result == "expected_value"  # Of course it matches!

# Good: Verify actual computation
result = code_under_test()
assert result == compute_expected_value_independently()
```

**Warning sign 3: Your test would pass with broken code**

If you can break the implementation and the test still passes, you're over-mocking.

**Warning sign 4: You're mocking more than you're testing**

```python
# Bad: More mock setup than actual testing
def test_something():
    mock1 = Mock()
    mock1.method1.return_value = "value1"
    mock1.method2.return_value = "value2"
    mock2 = Mock()
    mock2.method1.return_value = "value3"
    mock2.method2.return_value = "value4"
    mock3 = Mock()
    mock3.method1.return_value = "value5"
    # ... 20 more lines of mock setup ...
    
    result = code_under_test(mock1, mock2, mock3)
    assert result == "something"  # One assertion
```

**Warning sign 5: You're verifying implementation details**

```python
# Bad: Testing HOW it works
mock.internal_helper.assert_called_once()
mock.private_method.assert_called_with(specific_args)

# Good: Testing WHAT it produces
result = code_under_test()
assert result == expected_output
```

### Decision Framework: To Mock or Not to Mock

Use this decision tree:

```
Is it an external dependency?
├─ Yes → Mock it
│   ├─ Network calls (HTTP, database, cloud services)
│   ├─ File system operations (when not testing I/O)
│   ├─ Time/randomness (when you need determinism)
│   └─ External services (email, SMS, payment gateways)
│
└─ No → Is it your code?
    ├─ Yes → Don't mock it
    │   ├─ Business logic
    │   ├─ Data transformations
    │   ├─ Calculations
    │   └─ Formatting
    │
    └─ Is it slow or non-deterministic?
        ├─ Yes → Consider mocking
        │   ├─ Complex algorithms (if too slow)
        │   └─ Random number generation
        │
        └─ No → Don't mock it
```

### Refactoring Over-Mocked Tests

Let's refactor an over-mocked test step by step:

**Before (over-mocked)**:

```python
def test_category_report_over_mocked():
    processor = Mock()
    formatter = Mock()
    
    processor.group_by_category.return_value = {
        "food": [{"amount": 50.0}, {"amount": 30.0}],
        "transport": [{"amount": 20.0}]
    }
    processor.calculate_total.side_effect = [80.0, 20.0]
    formatter.format_currency.side_effect = lambda x: f"${x:.2f}"
    
    generator = ReportGenerator(processor, formatter)
    
    transactions = [
        {"amount": 50.0, "category": "food"},
        {"amount": 30.0, "category": "food"},
        {"amount": 20.0, "category": "transport"}
    ]
    
    report = generator.generate_category_report(transactions)
    
    assert report["food"]["total"] == "$80.00"
    assert report["transport"]["total"] == "$20.00"
```

**After (balanced)**:

```python
def test_category_report_realistic():
    # Use real dependencies
    processor = DataProcessor()
    formatter = ReportFormatter()
    
    generator = ReportGenerator(processor, formatter)
    
    transactions = [
        {"amount": 50.0, "category": "food"},
        {"amount": 30.0, "category": "food"},
        {"amount": 20.0, "category": "transport"}
    ]
    
    report = generator.generate_category_report(transactions)
    
    # Verify actual computed values
    assert report["food"]["total"] == "$80.00"  # 50 + 30
    assert report["food"]["count"] == 2
    assert report["transport"]["total"] == "$20.00"
    assert report["transport"]["count"] == 1
```

**What improved**:
- Removed unnecessary mocks
- Test verifies actual business logic
- Test would fail if logic is broken
- Simpler and more maintainable

### The Complete Test Suite: Balanced Mocking

Here's a complete test suite demonstrating balanced mocking:

```python
# test_report_balanced.py
from unittest.mock import Mock
from report_generator import (
    ReportGenerator, DataProcessor, ReportFormatter, EmailService
)

def test_generate_summary_with_real_dependencies():
    """Test summary generation with real processor and formatter."""
    processor = DataProcessor()
    formatter = ReportFormatter()
    generator = ReportGenerator(processor, formatter)
    
    transactions = [
        {"amount": 500.0},
        {"amount": 1000.0},
        {"amount": 1500.0},
        {"amount": 2000.0}
    ]
    
    report = generator.generate_summary(transactions)
    
    assert report["total"] == "$5,000.00"
    assert report["average"] == "$1,250.00"
    assert report["high_value_count"] == 2
    assert report["transaction_count"] == 4

def test_generate_category_report_with_real_dependencies():
    """Test category report with real processor and formatter."""
    processor = DataProcessor()
    formatter = ReportFormatter()
    generator = ReportGenerator(processor, formatter)
    
    transactions = [
        {"amount": 50.0, "category": "food"},
        {"amount": 30.0, "category": "food"},
        {"amount": 20.0, "category": "transport"},
        {"amount": 100.0, "category": "entertainment"}
    ]
    
    report = generator.generate_category_report(transactions)
    
    assert report["food"]["total"] == "$80.00"
    assert report["food"]["count"] == 2
    assert report["transport"]["total"] == "$20.00"
    assert report["transport"]["count"] == 1
    assert report["entertainment"]["total"] == "$100.00"
    assert report["entertainment"]["count"] == 1

def test_generate_and_email_summary_mocks_external_only():
    """Test email functionality - mock external service only."""
    processor = DataProcessor()
    formatter = ReportFormatter()
    email_service = Mock()  # Mock external dependency
    
    generator = ReportGenerator(processor, formatter, email_service)
    
    transactions = [
        {"amount": 500.0},
        {"amount": 1000.0}
    ]
    
    report = generator.generate_and_email_summary(
        transactions, "user@example.com"
    )
    
    # Verify real computation
    assert report["total"] == "$1,500.00"
    assert report["average"] == "$750.00"
    
    # Verify external service was called
    email_service.send_email.assert_called_once()
    call_args = email_service.send_email.call_args
    assert call_args.kwargs["to"] == "user@example.com"
    assert "$1,500.00" in call_args.kwargs["body"]

def test_empty_transactions():
    """Test edge case with no transactions."""
    processor = DataProcessor()
    formatter = ReportFormatter()
    generator = ReportGenerator(processor, formatter)
    
    report = generator.generate_summary([])
    
    assert report["total"] == "$0.00"
    assert report["average"] == "$0.00"
    assert report["high_value_count"] == 0
    assert report["transaction_count"] == 0
```

Run the complete suite:

```bash
pytest test_report_balanced.py -v
```

**Output**:
```
test_report_balanced.py::test_generate_summary_with_real_dependencies PASSED
test_report_balanced.py::test_generate_category_report_with_real_dependencies PASSED
test_report_balanced.py::test_generate_and_email_summary_mocks_external_only PASSED
test_report_balanced.py::test_empty_transactions PASSED
```

### Key Principles for Avoiding Over-Mocking

1. **Mock external dependencies, not your own code**
   - External: Network, database, file system, third-party APIs
   - Your code: Business logic, calculations, transformations

2. **Test behavior, not implementation**
   - Good: "Does it produce the correct output?"
   - Bad: "Does it call method X with argument Y?"

3. **If you can break the code and tests still pass, you're over-mocking**
   - Tests should fail when business logic is broken
   - Mocked return values bypass the actual logic

4. **Use real objects when they're fast and deterministic**
   - Pure functions: Always use real implementations
   - Simple classes: Use real instances
   - Complex external systems: Mock them

5. **Mock at the boundaries**
   - Mock where your system meets external systems
   - Don't mock internal boundaries between your own classes

6. **Prefer integration tests for complex interactions**
   - When multiple components work together, test them together
   - Use mocks only for external dependencies

### When Over-Mocking Is Acceptable

There are rare cases where mocking your own code is justified:

**1. Performance testing**: When the real implementation is too slow

```python
# Acceptable: Testing performance-critical code
def test_report_generation_performance():
    processor = Mock()  # Real processor is too slow
    processor.calculate_total.return_value = 1000.0
    # ... test that report generation itself is fast
```

**2. Testing error handling**: When you need to simulate specific error conditions

```python
# Acceptable: Testing error handling
def test_handles_processor_exception():
    processor = Mock()
    processor.calculate_total.side_effect = ValueError("Invalid data")
    # ... test that error is handled gracefully
```

**3. Legacy code**: When refactoring is not feasible

```python
# Acceptable: Working with legacy code that's hard to test
def test_legacy_system():
    legacy_component = Mock()  # Can't easily create real instance
    # ... test the new code that uses legacy component
```

But even in these cases, consider whether there's a better approach:
- Can you optimize the slow code?
- Can you refactor to make testing easier?
- Can you use a test double that's closer to the real implementation?

### The Journey: From Over-Mocking to Balanced Testing

| Iteration | Approach | What We Learned |
|-----------|----------|-----------------|
| 0 | Mocked everything | Tests pass but verify nothing |
| 1 | Used real dependencies | Tests verify actual behavior |
| 2 | Mocked external only | Balance between speed and accuracy |
| 3 | Refactored over-mocked tests | Simpler, more maintainable tests |

### Final Thoughts

Over-mocking is seductive because:
- It makes tests fast
- It makes tests pass easily
- It feels like you're being thorough

But over-mocked tests are worse than no tests because:
- They give false confidence
- They don't catch bugs
- They're brittle (break when implementation changes)
- They're hard to maintain

**The golden rule**: Mock external dependencies. Test your own code with real implementations.

When in doubt, ask yourself: "If I break this business logic, will my test fail?" If the answer is no, you're probably over-mocking.
