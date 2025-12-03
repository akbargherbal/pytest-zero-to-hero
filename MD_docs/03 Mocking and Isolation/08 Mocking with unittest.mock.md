# Chapter 8: Mocking with unittest.mock

## Why Mock?

## Why Mock?

Testing in isolation is one of the fundamental principles of effective unit testing. When you test a function, you want to verify *that specific function's behavior*—not the behavior of every external system it touches. But real-world code rarely exists in isolation. Functions call APIs, query databases, send emails, read files, and interact with countless external dependencies.

Consider this payment processing function:

```python
# payment_processor.py
import requests
from datetime import datetime

def process_payment(card_number, amount, currency="USD"):
    """Process a payment through external payment gateway."""
    response = requests.post(
        "https://api.payment-gateway.com/charge",
        json={
            "card": card_number,
            "amount": amount,
            "currency": currency,
            "timestamp": datetime.now().isoformat()
        },
        timeout=30
    )
    
    if response.status_code == 200:
        transaction_id = response.json()["transaction_id"]
        return {
            "success": True,
            "transaction_id": transaction_id,
            "amount": amount
        }
    else:
        return {
            "success": False,
            "error": response.json().get("error", "Unknown error")
        }
```

How do you test this function? If you run it directly in your tests, you face several problems:

### The Problems with Testing Real Dependencies

**Problem 1: External Service Dependency**

Your tests require a live payment gateway API. This means:
- Tests fail when your internet connection drops
- Tests fail when the API is down for maintenance
- Tests fail when the API changes its behavior
- You can't run tests offline or in isolated environments

**Problem 2: Side Effects**

Every test run charges real credit cards. This means:
- You need test credit card numbers that work with the real API
- You might accidentally charge real money
- You create transaction records in production systems
- You can't test error scenarios without triggering real failures

**Problem 3: Speed**

Network calls are slow. A single API call might take 500ms-2000ms. If you have 100 tests that each make API calls, your test suite takes minutes instead of seconds.

**Problem 4: Unpredictability**

External services are inherently unpredictable:
- Network latency varies
- API responses might differ based on time of day, rate limits, or server load
- You can't reliably test edge cases (what happens when the API returns a 503?)
- Tests become "flaky"—passing sometimes, failing other times

### The Solution: Mocking

Mocking solves all these problems by replacing external dependencies with controlled substitutes. Instead of making a real HTTP request, your test uses a fake `requests.post()` that returns exactly what you tell it to return.

With mocking, you can:
- Test without network access
- Run tests in milliseconds instead of seconds
- Simulate any response scenario (success, failure, timeout, malformed data)
- Verify that your code calls external services correctly
- Test in complete isolation

### What Mocking Is NOT

Before we dive deeper, let's clarify what mocking is *not*:

**Mocking is not cheating.** You're not skipping tests—you're testing your code's logic in isolation from external systems.

**Mocking is not testing fake behavior.** You're testing real code with controlled inputs. The logic inside `process_payment()` executes normally; only the external call is replaced.

**Mocking is not a replacement for integration tests.** You still need tests that verify your code works with real external systems. Mocking is for *unit* tests—testing individual components in isolation.

### When to Mock

Mock external dependencies when:
- The dependency is outside your control (APIs, databases, file systems)
- The dependency is slow (network calls, disk I/O)
- The dependency has side effects (sending emails, charging cards, deleting data)
- You need to test error scenarios that are hard to trigger naturally
- You want fast, reliable, repeatable tests

Don't mock when:
- Testing the integration between your code and the external system (use integration tests)
- The "dependency" is a simple, pure function in your own codebase
- Mocking would make the test more complex than the code being tested

### The Path Ahead

In this chapter, we'll build a complete understanding of mocking by:

1. Understanding what mocks are and how they work
2. Creating simple mocks to replace functions
3. Using `@patch` to intercept function calls
4. Verifying that mocked functions were called correctly
5. Controlling mock behavior with return values and side effects
6. Integrating mocks with pytest fixtures for reusable test infrastructure

We'll use the `process_payment()` function as our anchor example, progressively refining our tests to handle increasingly complex scenarios. Each iteration will expose a limitation, demonstrate the failure, and introduce the technique that solves it.

## What Is a Mock?

## What Is a Mock?

A mock is a programmable substitute for a real object. It looks like the real thing, responds like the real thing, but is completely under your control. Think of it as a stunt double for your dependencies—it stands in during testing so the real dependency doesn't have to show up.

### The Anatomy of a Mock

Let's start by examining what a mock object actually is. We'll create one manually to understand its structure before using Python's built-in mocking tools.

```python
# test_mock_basics.py
from unittest.mock import Mock

def test_mock_is_callable():
    """A mock can be called like a function."""
    mock_function = Mock()
    
    # You can call it with any arguments
    result = mock_function("any", "arguments", keyword="work")
    
    # By default, it returns another Mock
    assert isinstance(result, Mock)
    print(f"Called mock, got: {result}")
```

Run this test to see the basic behavior:

```bash
pytest test_mock_basics.py::test_mock_is_callable -v -s
```

**Output:**
```
test_mock_basics.py::test_mock_is_callable PASSED
Called mock, got: <Mock name='mock()' id='140234567890'>
```

The mock accepted any arguments and returned another mock. This is the fundamental behavior: **mocks are infinitely flexible**. They accept any call, return mocks for any attribute access, and never complain.

### Configuring Mock Return Values

The default behavior (returning another mock) isn't useful for testing. We need mocks to return specific values that our code expects.

```python
def test_mock_with_return_value():
    """Configure what a mock returns."""
    mock_function = Mock(return_value=42)
    
    result = mock_function("any", "arguments")
    
    assert result == 42
    print(f"Mock returned: {result}")
```

Now the mock returns exactly what we configured. This is how we simulate successful API responses, database queries, or any other function call.

### Mocks Record Their Interactions

The real power of mocks is that they remember how they were used. This lets you verify that your code called dependencies correctly.

```python
def test_mock_records_calls():
    """Mocks remember how they were called."""
    mock_function = Mock(return_value="success")
    
    # Call the mock
    result = mock_function("test_arg", keyword="test_value")
    
    # Verify it was called
    assert mock_function.called
    assert mock_function.call_count == 1
    
    # Verify it was called with specific arguments
    mock_function.assert_called_once_with("test_arg", keyword="test_value")
    
    print(f"Mock was called: {mock_function.called}")
    print(f"Call count: {mock_function.call_count}")
    print(f"Called with: {mock_function.call_args}")
```

**Output:**
```
Mock was called: True
Call count: 1
Called with: call('test_arg', keyword='test_value')
```

The mock tracked:
- Whether it was called at all (`called`)
- How many times it was called (`call_count`)
- What arguments it received (`call_args`)

### Mocks Have Attributes Too

Mocks can simulate objects with attributes and methods, not just standalone functions.

```python
def test_mock_with_attributes():
    """Mocks can have attributes and methods."""
    mock_api_response = Mock()
    mock_api_response.status_code = 200
    mock_api_response.json.return_value = {"transaction_id": "txn_12345"}
    
    # Use it like a real response object
    assert mock_api_response.status_code == 200
    data = mock_api_response.json()
    assert data["transaction_id"] == "txn_12345"
    
    print(f"Status: {mock_api_response.status_code}")
    print(f"JSON data: {data}")
```

This mock simulates an HTTP response object with a `status_code` attribute and a `json()` method. This is exactly what `requests.post()` returns, so we can use this mock to test code that processes API responses.

### The Mock Hierarchy

When you access an attribute on a mock that hasn't been configured, you get another mock:

```python
def test_mock_attribute_chain():
    """Unconfigured attributes return new mocks."""
    mock_obj = Mock()
    
    # Access a chain of attributes
    result = mock_obj.some.deeply.nested.attribute()
    
    # Each step returns a mock
    assert isinstance(mock_obj.some, Mock)
    assert isinstance(mock_obj.some.deeply, Mock)
    assert isinstance(result, Mock)
    
    print(f"mock_obj.some: {mock_obj.some}")
    print(f"mock_obj.some.deeply: {mock_obj.some.deeply}")
    print(f"Final result: {result}")
```

This "infinite mock" behavior is useful for mocking complex objects without configuring every possible attribute. However, it can also hide bugs—if you misspell an attribute name, the mock won't complain.

### Comparing Mocks to Real Objects

Let's see the difference between a real object and a mock side by side:

```python
import requests

def test_real_vs_mock_response():
    """Compare a real response object to a mock."""
    # Create a mock that mimics requests.Response
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"message": "success"}
    mock_response.text = '{"message": "success"}'
    
    # The mock behaves like a real response
    assert mock_response.status_code == 200
    assert mock_response.json() == {"message": "success"}
    
    # But it's not the same type
    print(f"Mock type: {type(mock_response)}")
    print(f"Real response type: {type(requests.Response())}")
    print(f"Mock status_code: {mock_response.status_code}")
    print(f"Mock json(): {mock_response.json()}")
```

**Output:**
```
Mock type: <class 'unittest.mock.Mock'>
Real response type: <class 'requests.models.Response'>
Mock status_code: 200
Mock json(): {'message': 'success'}
```

The mock isn't a `requests.Response` object, but it has the same interface. For testing purposes, that's all we need.

### When Mocks Fail: The Spec Parameter

The infinite flexibility of mocks can be dangerous. If you misspell a method name, the mock won't tell you:

```python
def test_mock_without_spec_hides_errors():
    """Mocks accept any attribute access, even typos."""
    mock_response = Mock()
    mock_response.status_code = 200
    
    # Typo: should be .json() not .jsno()
    # The mock doesn't complain!
    result = mock_response.jsno()
    
    # This passes, but it shouldn't
    assert isinstance(result, Mock)
    print(f"Typo returned: {result}")
```

This test passes, but it's testing the wrong thing. The real code calls `.json()`, not `.jsno()`. The mock's flexibility hid our mistake.

The solution is to use `spec` to restrict the mock to match a real object's interface:

```python
def test_mock_with_spec_catches_errors():
    """Spec restricts mocks to real object interfaces."""
    mock_response = Mock(spec=requests.Response)
    mock_response.status_code = 200
    mock_response.json.return_value = {"message": "success"}
    
    # This works - json() exists on Response
    result = mock_response.json()
    assert result == {"message": "success"}
    
    # This would raise AttributeError - jsno() doesn't exist
    # Uncomment to see the error:
    # mock_response.jsno()
    
    print(f"Valid method call succeeded: {result}")
```

With `spec=requests.Response`, the mock only allows attributes and methods that exist on the real `Response` class. Typos now raise `AttributeError` immediately.

### Summary: What We've Learned

A mock is:
- A programmable substitute for real objects
- Callable with any arguments (by default)
- Configurable to return specific values
- Capable of recording all interactions
- Able to simulate attributes and methods
- Infinitely flexible (which can be dangerous)
- Constrainable with `spec` to match real interfaces

In the next section, we'll use these building blocks to create mocks for our `process_payment()` function, replacing the real `requests.post()` call with a controlled substitute.

## Creating Mocks with Mock()

## Creating Mocks with Mock()

Now that we understand what mocks are, let's use them to test real code. We'll return to our payment processing function and progressively build a complete test suite using mocks.

### Phase 1: The Reference Implementation

Here's our payment processor again, which will serve as our anchor example throughout this chapter:

```python
# payment_processor.py
import requests
from datetime import datetime

def process_payment(card_number, amount, currency="USD"):
    """Process a payment through external payment gateway."""
    response = requests.post(
        "https://api.payment-gateway.com/charge",
        json={
            "card": card_number,
            "amount": amount,
            "currency": currency,
            "timestamp": datetime.now().isoformat()
        },
        timeout=30
    )
    
    if response.status_code == 200:
        transaction_id = response.json()["transaction_id"]
        return {
            "success": True,
            "transaction_id": transaction_id,
            "amount": amount
        }
    else:
        return {
            "success": False,
            "error": response.json().get("error", "Unknown error")
        }
```

Our goal is to test this function without making real HTTP requests. We need to replace `requests.post()` with a mock that returns controlled responses.

### Iteration 0: The Naive Approach (No Mocking)

Let's first try testing without mocks to see why we need them:

```python
# test_payment_naive.py
from payment_processor import process_payment

def test_successful_payment_naive():
    """Attempt to test without mocking."""
    result = process_payment("4111111111111111", 100.00)
    
    assert result["success"] == True
    assert "transaction_id" in result
    assert result["amount"] == 100.00
```

Run this test:

```bash
pytest test_payment_naive.py -v
```

**Output:**
```
test_payment_naive.py::test_successful_payment_naive FAILED

================================ FAILURES =================================
________________________ test_successful_payment_naive ________________________

    def test_successful_payment_naive():
        """Attempt to test without mocking."""
>       result = process_payment("4111111111111111", 100.00)

test_payment_naive.py:5: 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _

    response = requests.post(
        "https://api.payment-gateway.com/charge",
        ...
    )

E   requests.exceptions.ConnectionError: HTTPSConnectionPool(host='api.payment-gateway.com', port=443): 
    Max retries exceeded with url: /charge (Caused by NewConnectionError(
    '<urllib3.connection.HTTPSConnection object at 0x7f8b8c0d4d90>: 
    Failed to establish a new connection: [Errno -2] Name or service not known'))
```

### Diagnostic Analysis: Reading the Failure

**The complete output tells us**:

1. **The summary line**: `FAILED test_payment_naive.py::test_successful_payment_naive`
   - The test failed during execution, not during collection

2. **The traceback**:
   - The failure occurred inside `process_payment()` when calling `requests.post()`
   - The error is `requests.exceptions.ConnectionError`

3. **The error details**:
   - "Failed to establish a new connection"
   - "Name or service not known"
   - This means the test tried to make a real HTTP request to a non-existent domain

**Root cause identified**: The test is attempting real network communication.

**Why the current approach can't solve this**: We can't make the domain exist or guarantee network availability in tests.

**What we need**: A way to intercept the `requests.post()` call and return a fake response without touching the network.

### Iteration 1: Manual Mock Injection

Our first attempt will manually pass a mock into the function. This requires modifying the function to accept an optional dependency:

```python
# payment_processor_injectable.py
import requests
from datetime import datetime

def process_payment(card_number, amount, currency="USD", http_client=None):
    """Process a payment through external payment gateway.
    
    Args:
        http_client: Optional HTTP client for testing (defaults to requests)
    """
    if http_client is None:
        http_client = requests
    
    response = http_client.post(
        "https://api.payment-gateway.com/charge",
        json={
            "card": card_number,
            "amount": amount,
            "currency": currency,
            "timestamp": datetime.now().isoformat()
        },
        timeout=30
    )
    
    if response.status_code == 200:
        transaction_id = response.json()["transaction_id"]
        return {
            "success": True,
            "transaction_id": transaction_id,
            "amount": amount
        }
    else:
        return {
            "success": False,
            "error": response.json().get("error", "Unknown error")
        }
```

Now we can inject a mock:

```python
# test_payment_injectable.py
from unittest.mock import Mock
from payment_processor_injectable import process_payment

def test_successful_payment_with_injection():
    """Test by injecting a mock HTTP client."""
    # Create a mock HTTP client
    mock_client = Mock()
    
    # Configure the mock response
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"transaction_id": "txn_12345"}
    
    # Make the mock client return our mock response
    mock_client.post.return_value = mock_response
    
    # Inject the mock
    result = process_payment("4111111111111111", 100.00, http_client=mock_client)
    
    # Verify the result
    assert result["success"] == True
    assert result["transaction_id"] == "txn_12345"
    assert result["amount"] == 100.00
    
    # Verify the mock was called correctly
    mock_client.post.assert_called_once()
    call_args = mock_client.post.call_args
    assert call_args.args[0] == "https://api.payment-gateway.com/charge"
    assert call_args.kwargs["json"]["amount"] == 100.00
```

Run this test:

```bash
pytest test_payment_injectable.py::test_successful_payment_with_injection -v
```

**Output:**
```
test_payment_injectable.py::test_successful_payment_with_injection PASSED
```

Success! The test passes without making network requests. Let's analyze what happened:

1. We created a mock HTTP client with `Mock()`
2. We created a mock response object with the expected attributes
3. We configured `mock_client.post` to return our mock response
4. We injected the mock into the function
5. The function executed normally, using our mock instead of real `requests`
6. We verified both the result and that the mock was called correctly

### Current Limitation: Requires Code Modification

This approach works, but it has a significant drawback: **we had to modify the production code to accept an optional dependency**. The `http_client` parameter exists solely for testing. This is called "dependency injection," and while it's a valid pattern, it's not always desirable:

- It clutters the function signature
- It exposes internal implementation details
- It requires all callers to know about this testing parameter
- It doesn't work for code you don't control (third-party libraries)

**What we need**: A way to replace `requests.post()` without modifying the function signature.

### Iteration 2: Direct Mock Assignment

Python's dynamic nature allows us to replace module-level objects at runtime:

```python
# test_payment_direct_mock.py
from unittest.mock import Mock
import payment_processor

def test_successful_payment_direct_replacement():
    """Test by directly replacing requests.post."""
    # Save the original
    original_post = payment_processor.requests.post
    
    try:
        # Create and configure mock response
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {"transaction_id": "txn_12345"}
        
        # Create mock post function
        mock_post = Mock(return_value=mock_response)
        
        # Replace requests.post in the payment_processor module
        payment_processor.requests.post = mock_post
        
        # Now test the function
        result = payment_processor.process_payment("4111111111111111", 100.00)
        
        assert result["success"] == True
        assert result["transaction_id"] == "txn_12345"
        
        # Verify the mock was called
        mock_post.assert_called_once()
        
    finally:
        # Restore the original
        payment_processor.requests.post = original_post
```

Run this test:

```bash
pytest test_payment_direct_mock.py::test_successful_payment_direct_replacement -v
```

**Output:**
```
test_payment_direct_mock.py::test_successful_payment_direct_replacement PASSED
```

This works! We replaced `requests.post` directly in the module where it's used. The function doesn't need modification.

### Current Limitation: Manual Cleanup Required

This approach has problems:

1. **Verbose**: Lots of boilerplate for saving, replacing, and restoring
2. **Error-prone**: If the test fails before the `finally` block, cleanup might not happen
3. **Repetitive**: Every test needs this same setup/teardown pattern

**What we need**: A cleaner way to temporarily replace objects that automatically handles cleanup.

This is exactly what `@patch` provides, which we'll explore in the next section.

### Summary: What We've Learned

We've seen three approaches to mocking:

1. **No mocking**: Tests fail due to external dependencies
2. **Dependency injection**: Works but requires code modification
3. **Direct replacement**: Works without code modification but requires manual cleanup

Key insights:
- Mocks simulate objects by providing the same interface
- Configure mocks with `return_value` to control what they return
- Mocks record calls so you can verify interactions
- Direct replacement works but needs careful cleanup

The next section introduces `@patch`, which automates the replacement and cleanup process.

## Patching Functions with @patch

## Patching Functions with @patch

The `@patch` decorator automates the process of replacing objects during tests and restoring them afterward. It's the standard way to mock in Python testing.

### Understanding Patch Targets

Before using `@patch`, you must understand *where* to patch. This is the most common source of confusion with mocking.

**The Golden Rule**: Patch where the object is *used*, not where it's *defined*.

Consider our payment processor:

```python
# payment_processor.py
import requests  # requests is defined in the requests module

def process_payment(card_number, amount, currency="USD"):
    response = requests.post(...)  # requests is used here
```

When `process_payment()` executes, it looks up `requests` in its own module's namespace (`payment_processor.requests`), not in the original `requests` module. Therefore, we must patch `payment_processor.requests`, not `requests.requests`.

### Iteration 3: Basic @patch Usage

Let's rewrite our test using `@patch`:

```python
# test_payment_with_patch.py
from unittest.mock import patch, Mock
from payment_processor import process_payment

@patch('payment_processor.requests.post')
def test_successful_payment_with_patch(mock_post):
    """Test using @patch decorator."""
    # Configure the mock response
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"transaction_id": "txn_12345"}
    
    # Configure mock_post to return our response
    mock_post.return_value = mock_response
    
    # Call the function
    result = process_payment("4111111111111111", 100.00)
    
    # Verify the result
    assert result["success"] == True
    assert result["transaction_id"] == "txn_12345"
    assert result["amount"] == 100.00
    
    # Verify the mock was called correctly
    mock_post.assert_called_once()
    call_args = mock_post.call_args
    assert call_args.kwargs["json"]["card"] == "4111111111111111"
    assert call_args.kwargs["json"]["amount"] == 100.00
```

Run this test:

```bash
pytest test_payment_with_patch.py::test_successful_payment_with_patch -v
```

**Output:**
```
test_payment_with_patch.py::test_successful_payment_with_patch PASSED
```

Let's break down what happened:

1. `@patch('payment_processor.requests.post')` replaces `requests.post` in the `payment_processor` module
2. The replacement (a `Mock` object) is passed as the `mock_post` parameter
3. We configure `mock_post` to return our mock response
4. The test runs with the mock in place
5. After the test completes, `@patch` automatically restores the original `requests.post`

### Comparing Before and After

**Before (manual replacement)**:
```python
original = payment_processor.requests.post
try:
    payment_processor.requests.post = mock_post
    # test code
finally:
    payment_processor.requests.post = original
```

**After (@patch)**:
```python
@patch('payment_processor.requests.post')
def test_something(mock_post):
    # test code
```

The decorator handles all the setup and teardown automatically.

### Iteration 4: Testing Error Scenarios

Now let's test what happens when the payment gateway returns an error. We'll use the same mocking approach but configure a different response:

```python
@patch('payment_processor.requests.post')
def test_failed_payment_with_patch(mock_post):
    """Test payment failure scenario."""
    # Configure mock to return error response
    mock_response = Mock()
    mock_response.status_code = 400
    mock_response.json.return_value = {"error": "Invalid card number"}
    
    mock_post.return_value = mock_response
    
    # Call the function
    result = process_payment("0000000000000000", 100.00)
    
    # Verify error handling
    assert result["success"] == False
    assert result["error"] == "Invalid card number"
    
    # Verify the request was still made
    mock_post.assert_called_once()
```

Run this test:

```bash
pytest test_payment_with_patch.py::test_failed_payment_with_patch -v
```

**Output:**
```
test_payment_with_patch.py::test_failed_payment_with_patch PASSED
```

With mocking, we can easily test error scenarios that would be difficult or impossible to trigger with a real API.

### Iteration 5: Multiple Patches

What if we also want to control the timestamp? Our function calls `datetime.now()`, which returns the current time. This makes tests non-deterministic—the exact timestamp changes with each run.

Let's patch both `requests.post` and `datetime.now`:

```python
from datetime import datetime

@patch('payment_processor.datetime')
@patch('payment_processor.requests.post')
def test_payment_with_fixed_timestamp(mock_post, mock_datetime):
    """Test with controlled timestamp."""
    # Configure datetime mock
    fixed_time = datetime(2024, 1, 15, 10, 30, 0)
    mock_datetime.now.return_value = fixed_time
    
    # Configure response mock
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"transaction_id": "txn_12345"}
    mock_post.return_value = mock_response
    
    # Call the function
    result = process_payment("4111111111111111", 100.00)
    
    # Verify the result
    assert result["success"] == True
    
    # Verify the timestamp was used correctly
    call_args = mock_post.call_args
    expected_timestamp = "2024-01-15T10:30:00"
    assert call_args.kwargs["json"]["timestamp"] == expected_timestamp
```

**Important**: When stacking `@patch` decorators, they are applied **bottom-to-top**, but parameters are passed **top-to-bottom**:

```python
@patch('module.second')  # This becomes the second parameter
@patch('module.first')   # This becomes the first parameter
def test_something(mock_first, mock_second):
    pass
```

Run this test:

```bash
pytest test_payment_with_patch.py::test_payment_with_fixed_timestamp -v
```

**Output:**
```
test_payment_with_patch.py::test_payment_with_fixed_timestamp PASSED
```

### Iteration 6: Patch as Context Manager

Sometimes you need to patch only part of a test. Use `patch()` as a context manager:

```python
from unittest.mock import patch

def test_payment_with_context_manager():
    """Use patch as a context manager for fine-grained control."""
    # Setup code without mocking
    card_number = "4111111111111111"
    amount = 100.00
    
    # Only mock during the actual call
    with patch('payment_processor.requests.post') as mock_post:
        # Configure mock
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {"transaction_id": "txn_12345"}
        mock_post.return_value = mock_response
        
        # Call function
        result = process_payment(card_number, amount)
        
        # Assertions inside the context
        assert result["success"] == True
        mock_post.assert_called_once()
    
    # Outside the context, requests.post is restored
    # (though we can't easily demonstrate this in a test)
```

The context manager form is useful when:
- You need to patch only part of a test
- You want to make multiple calls with different mock configurations
- You're patching in setup code that isn't a test function

### Iteration 7: Patching Object Attributes

You can patch attributes, not just functions:

```python
# config.py
API_ENDPOINT = "https://api.payment-gateway.com/charge"
API_TIMEOUT = 30

# payment_processor_with_config.py
import requests
from datetime import datetime
import config

def process_payment(card_number, amount, currency="USD"):
    """Process payment using configuration."""
    response = requests.post(
        config.API_ENDPOINT,  # Use config value
        json={
            "card": card_number,
            "amount": amount,
            "currency": currency,
            "timestamp": datetime.now().isoformat()
        },
        timeout=config.API_TIMEOUT  # Use config value
    )
    # ... rest of function
```

```python
# test_payment_with_config.py
@patch('payment_processor_with_config.config.API_ENDPOINT', 'https://test-api.example.com')
@patch('payment_processor_with_config.config.API_TIMEOUT', 5)
@patch('payment_processor_with_config.requests.post')
def test_payment_uses_config(mock_post, mock_timeout, mock_endpoint):
    """Test that function uses configuration values."""
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"transaction_id": "txn_12345"}
    mock_post.return_value = mock_response
    
    result = process_payment("4111111111111111", 100.00)
    
    # Verify the mocked config values were used
    call_args = mock_post.call_args
    assert call_args.args[0] == 'https://test-api.example.com'
    assert call_args.kwargs['timeout'] == 5
```

### Common Patching Mistakes

#### Mistake 1: Patching in the Wrong Place

**Wrong**:
```python
@patch('requests.post')  # Patches requests module, not where it's used
def test_payment(mock_post):
    result = process_payment("4111111111111111", 100.00)
```

This doesn't work because `process_payment` looks up `requests` in its own namespace.

**Right**:
```python
@patch('payment_processor.requests.post')  # Patches where it's used
def test_payment(mock_post):
    result = process_payment("4111111111111111", 100.00)
```

#### Mistake 2: Forgetting to Configure the Mock

**Wrong**:
```python
@patch('payment_processor.requests.post')
def test_payment(mock_post):
    # Forgot to configure mock_post!
    result = process_payment("4111111111111111", 100.00)
    # result will contain Mock objects, not real data
```

**Right**:
```python
@patch('payment_processor.requests.post')
def test_payment(mock_post):
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"transaction_id": "txn_12345"}
    mock_post.return_value = mock_response
    
    result = process_payment("4111111111111111", 100.00)
```

#### Mistake 3: Wrong Parameter Order with Multiple Patches

**Wrong**:
```python
@patch('module.second')
@patch('module.first')
def test_something(mock_second, mock_first):  # Wrong order!
    pass
```

**Right**:
```python
@patch('module.second')
@patch('module.first')
def test_something(mock_first, mock_second):  # Matches decorator order
    pass
```

### When to Use Each Form

**Use decorator form** when:
- The entire test needs the mock
- You're mocking the same thing across multiple tests
- You want clean, declarative test signatures

**Use context manager form** when:
- Only part of the test needs the mock
- You need different mock configurations in the same test
- You're patching in non-test code (fixtures, setup functions)

**Use manual replacement** when:
- You need very fine-grained control
- You're doing something unusual that `@patch` doesn't support
- Never, actually—`@patch` handles almost everything

### Summary: The Power of @patch

`@patch` provides:
- Automatic setup and teardown
- Clean, declarative syntax
- Support for multiple patches
- Flexible usage (decorator or context manager)
- Correct restoration even when tests fail

The key insight: **patch where the object is used, not where it's defined**.

In the next section, we'll learn how to verify that mocked functions were called correctly using mock assertions.

## Common Mock Assertions (assert_called, assert_called_with)

## Common Mock Assertions (assert_called, assert_called_with)

Mocking isn't just about replacing dependencies—it's also about verifying that your code interacts with those dependencies correctly. Mock assertions let you check that functions were called with the right arguments, the right number of times, and in the right order.

### Why Verify Mock Calls?

Consider this scenario: your payment processor successfully returns a result, but you forgot to actually send the payment request. The test passes because the mock returns what you configured, but the real code would fail in production.

Mock assertions catch these bugs by verifying that your code actually called the mocked function.

### Iteration 8: Basic Call Verification

Let's start with the simplest assertion—checking if a mock was called at all:

```python
# test_mock_assertions.py
from unittest.mock import patch, Mock
from payment_processor import process_payment

@patch('payment_processor.requests.post')
def test_payment_makes_api_call(mock_post):
    """Verify that the function actually calls the API."""
    # Configure mock
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"transaction_id": "txn_12345"}
    mock_post.return_value = mock_response
    
    # Call function
    result = process_payment("4111111111111111", 100.00)
    
    # Verify the mock was called
    assert mock_post.called
    print(f"Mock was called: {mock_post.called}")
    print(f"Call count: {mock_post.call_count}")
```

Run this test:

```bash
pytest test_mock_assertions.py::test_payment_makes_api_call -v -s
```

**Output:**
```
test_mock_assertions.py::test_payment_makes_api_call PASSED
Mock was called: True
Call count: 1
```

The `called` attribute is `True` if the mock was called at least once. But this is a weak assertion—it doesn't verify *how* the mock was called.

### Iteration 9: Verifying Call Count

Let's verify the mock was called exactly once:

```python
@patch('payment_processor.requests.post')
def test_payment_calls_api_once(mock_post):
    """Verify the API is called exactly once."""
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"transaction_id": "txn_12345"}
    mock_post.return_value = mock_response
    
    result = process_payment("4111111111111111", 100.00)
    
    # Verify called exactly once
    mock_post.assert_called_once()
    
    # Alternative: check call_count directly
    assert mock_post.call_count == 1
```

`assert_called_once()` is more expressive than checking `call_count == 1`. It also provides better error messages when it fails.

### Iteration 10: Verifying Call Arguments

Now let's verify the mock was called with the correct arguments:

```python
@patch('payment_processor.requests.post')
def test_payment_sends_correct_data(mock_post):
    """Verify the API is called with correct arguments."""
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"transaction_id": "txn_12345"}
    mock_post.return_value = mock_response
    
    # Call with specific values
    result = process_payment("4111111111111111", 100.00, currency="EUR")
    
    # Verify the call arguments
    mock_post.assert_called_once_with(
        "https://api.payment-gateway.com/charge",
        json={
            "card": "4111111111111111",
            "amount": 100.00,
            "currency": "EUR",
            "timestamp": mock_post.call_args.kwargs["json"]["timestamp"]  # We'll verify this separately
        },
        timeout=30
    )
```

Run this test:

```bash
pytest test_mock_assertions.py::test_payment_sends_correct_data -v
```

**Output:**
```
test_mock_assertions.py::test_payment_sends_correct_data PASSED
```

`assert_called_once_with()` verifies:
- The mock was called exactly once
- The arguments match exactly

But there's a problem: we had to use `mock_post.call_args.kwargs["json"]["timestamp"]` because the timestamp changes with each call. This is awkward.

### Iteration 11: Partial Argument Verification

When you can't predict all arguments (like timestamps), verify only the parts you care about:

```python
@patch('payment_processor.requests.post')
def test_payment_sends_correct_data_partial(mock_post):
    """Verify specific arguments without checking everything."""
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"transaction_id": "txn_12345"}
    mock_post.return_value = mock_response
    
    result = process_payment("4111111111111111", 100.00, currency="EUR")
    
    # Get the actual call arguments
    call_args = mock_post.call_args
    
    # Verify positional arguments
    assert call_args.args[0] == "https://api.payment-gateway.com/charge"
    
    # Verify specific keyword arguments
    assert call_args.kwargs["timeout"] == 30
    
    # Verify JSON payload structure
    json_data = call_args.kwargs["json"]
    assert json_data["card"] == "4111111111111111"
    assert json_data["amount"] == 100.00
    assert json_data["currency"] == "EUR"
    assert "timestamp" in json_data  # Just verify it exists
    
    print(f"Call args: {call_args}")
    print(f"JSON data: {json_data}")
```

Run this test:

```bash
pytest test_mock_assertions.py::test_payment_sends_correct_data_partial -v -s
```

**Output:**
```
test_mock_assertions.py::test_payment_sends_correct_data_partial PASSED
Call args: call('https://api.payment-gateway.com/charge', json={'card': '4111111111111111', 'amount': 100.0, 'currency': 'EUR', 'timestamp': '2024-01-15T10:30:00.123456'}, timeout=30)
JSON data: {'card': '4111111111111111', 'amount': 100.0, 'currency': 'EUR', 'timestamp': '2024-01-15T10:30:00.123456'}
```

This approach is more flexible. We verify the important parts and ignore the unpredictable timestamp value.

### Understanding call_args

`call_args` is a `call` object with two attributes:
- `args`: Tuple of positional arguments
- `kwargs`: Dictionary of keyword arguments

You can access them like this:

```python
def test_understanding_call_args():
    """Demonstrate call_args structure."""
    mock_func = Mock()
    
    # Call with mixed arguments
    mock_func("pos1", "pos2", key1="val1", key2="val2")
    
    # Access positional arguments
    assert mock_func.call_args.args == ("pos1", "pos2")
    assert mock_func.call_args.args[0] == "pos1"
    
    # Access keyword arguments
    assert mock_func.call_args.kwargs == {"key1": "val1", "key2": "val2"}
    assert mock_func.call_args.kwargs["key1"] == "val1"
    
    # Alternative: unpack the call object
    args, kwargs = mock_func.call_args
    assert args == ("pos1", "pos2")
    assert kwargs == {"key1": "val1", "key2": "val2"}
    
    print(f"Full call_args: {mock_func.call_args}")
    print(f"Positional: {mock_func.call_args.args}")
    print(f"Keyword: {mock_func.call_args.kwargs}")
```

### Iteration 12: Verifying Multiple Calls

What if a function calls the mock multiple times? Use `call_args_list` to inspect all calls:

```python
# payment_processor_batch.py
import requests
from datetime import datetime

def process_batch_payments(payments):
    """Process multiple payments.
    
    Args:
        payments: List of (card_number, amount) tuples
    
    Returns:
        List of results
    """
    results = []
    for card_number, amount in payments:
        response = requests.post(
            "https://api.payment-gateway.com/charge",
            json={
                "card": card_number,
                "amount": amount,
                "currency": "USD",
                "timestamp": datetime.now().isoformat()
            },
            timeout=30
        )
        
        if response.status_code == 200:
            results.append({
                "success": True,
                "transaction_id": response.json()["transaction_id"]
            })
        else:
            results.append({
                "success": False,
                "error": response.json().get("error", "Unknown error")
            })
    
    return results
```

```python
# test_batch_payments.py
from unittest.mock import patch, Mock
from payment_processor_batch import process_batch_payments

@patch('payment_processor_batch.requests.post')
def test_batch_payment_multiple_calls(mock_post):
    """Verify multiple API calls for batch processing."""
    # Configure mock to return different responses
    mock_response1 = Mock()
    mock_response1.status_code = 200
    mock_response1.json.return_value = {"transaction_id": "txn_001"}
    
    mock_response2 = Mock()
    mock_response2.status_code = 200
    mock_response2.json.return_value = {"transaction_id": "txn_002"}
    
    # Return different responses for each call
    mock_post.side_effect = [mock_response1, mock_response2]
    
    # Process batch
    payments = [
        ("4111111111111111", 100.00),
        ("4222222222222222", 200.00)
    ]
    results = process_batch_payments(payments)
    
    # Verify two calls were made
    assert mock_post.call_count == 2
    
    # Verify first call
    first_call = mock_post.call_args_list[0]
    assert first_call.kwargs["json"]["card"] == "4111111111111111"
    assert first_call.kwargs["json"]["amount"] == 100.00
    
    # Verify second call
    second_call = mock_post.call_args_list[1]
    assert second_call.kwargs["json"]["card"] == "4222222222222222"
    assert second_call.kwargs["json"]["amount"] == 200.00
    
    print(f"Total calls: {mock_post.call_count}")
    print(f"First call: {first_call}")
    print(f"Second call: {second_call}")
```

Run this test:

```bash
pytest test_batch_payments.py::test_batch_payment_multiple_calls -v -s
```

**Output:**
```
test_batch_payments.py::test_batch_payment_multiple_calls PASSED
Total calls: 2
First call: call('https://api.payment-gateway.com/charge', json={'card': '4111111111111111', 'amount': 100.0, 'currency': 'USD', 'timestamp': '2024-01-15T10:30:00.123456'}, timeout=30)
Second call: call('https://api.payment-gateway.com/charge', json={'card': '4222222222222222', 'amount': 200.0, 'currency': 'USD', 'timestamp': '2024-01-15T10:30:00.234567'}, timeout=30)
```

`call_args_list` is a list of all `call` objects, indexed from 0.

### Iteration 13: Using assert_any_call

When you don't care about call order or count, just verify a specific call happened:

```python
@patch('payment_processor_batch.requests.post')
def test_batch_payment_includes_specific_call(mock_post):
    """Verify a specific call was made, regardless of order."""
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"transaction_id": "txn_001"}
    mock_post.return_value = mock_response
    
    payments = [
        ("4111111111111111", 100.00),
        ("4222222222222222", 200.00),
        ("4333333333333333", 300.00)
    ]
    results = process_batch_payments(payments)
    
    # Verify the second payment was processed
    # We don't care about the timestamp, so we can't use assert_called_with
    # Instead, check manually
    second_call = mock_post.call_args_list[1]
    assert second_call.kwargs["json"]["card"] == "4222222222222222"
    assert second_call.kwargs["json"]["amount"] == 200.00
```

### Iteration 14: Verifying No Calls

Sometimes you need to verify a mock was *not* called:

```python
# payment_processor_conditional.py
import requests
from datetime import datetime

def process_payment_if_valid(card_number, amount):
    """Only process payment if amount is positive."""
    if amount <= 0:
        return {"success": False, "error": "Invalid amount"}
    
    response = requests.post(
        "https://api.payment-gateway.com/charge",
        json={
            "card": card_number,
            "amount": amount,
            "currency": "USD",
            "timestamp": datetime.now().isoformat()
        },
        timeout=30
    )
    
    if response.status_code == 200:
        return {
            "success": True,
            "transaction_id": response.json()["transaction_id"]
        }
    else:
        return {
            "success": False,
            "error": response.json().get("error", "Unknown error")
        }
```

```python
# test_conditional_payment.py
from unittest.mock import patch
from payment_processor_conditional import process_payment_if_valid

@patch('payment_processor_conditional.requests.post')
def test_invalid_amount_skips_api_call(mock_post):
    """Verify API is not called for invalid amounts."""
    result = process_payment_if_valid("4111111111111111", -50.00)
    
    # Verify the result
    assert result["success"] == False
    assert result["error"] == "Invalid amount"
    
    # Verify the API was never called
    mock_post.assert_not_called()
    
    print(f"Mock called: {mock_post.called}")
    print(f"Call count: {mock_post.call_count}")
```

Run this test:

```bash
pytest test_conditional_payment.py::test_invalid_amount_skips_api_call -v -s
```

**Output:**
```
test_conditional_payment.py::test_invalid_amount_skips_api_call PASSED
Mock called: False
Call count: 0
```

`assert_not_called()` verifies the mock was never invoked. This is useful for testing conditional logic.

### Complete Mock Assertion Reference

Here's a summary of all mock assertion methods:

| Assertion | What It Checks |
|-----------|----------------|
| `mock.assert_called()` | Mock was called at least once |
| `mock.assert_called_once()` | Mock was called exactly once |
| `mock.assert_called_with(*args, **kwargs)` | Most recent call used these arguments |
| `mock.assert_called_once_with(*args, **kwargs)` | Called exactly once with these arguments |
| `mock.assert_any_call(*args, **kwargs)` | At least one call used these arguments |
| `mock.assert_not_called()` | Mock was never called |
| `mock.assert_has_calls(calls, any_order=False)` | Verify multiple specific calls |

### Common Assertion Mistakes

#### Mistake 1: Using assert_called_with After Multiple Calls

**Wrong**:
```python
mock_func("first")
mock_func("second")
mock_func.assert_called_with("first")  # Fails! Checks most recent call
```

**Right**:
```python
mock_func("first")
mock_func("second")
mock_func.assert_called_with("second")  # Checks most recent call
# OR
mock_func.assert_any_call("first")  # Checks any call
```

#### Mistake 2: Forgetting to Verify Calls

**Wrong**:
```python
@patch('module.func')
def test_something(mock_func):
    mock_func.return_value = "result"
    result = my_function()
    assert result == "expected"
    # Forgot to verify mock_func was called!
```

**Right**:
```python
@patch('module.func')
def test_something(mock_func):
    mock_func.return_value = "result"
    result = my_function()
    assert result == "expected"
    mock_func.assert_called_once()  # Verify the mock was used
```

#### Mistake 3: Over-Specifying Arguments

**Wrong**:
```python
# Fails if timestamp format changes slightly
mock_post.assert_called_once_with(
    "https://api.example.com",
    json={"timestamp": "2024-01-15T10:30:00.123456"}
)
```

**Right**:
```python
# Verify only what matters
call_args = mock_post.call_args
assert "timestamp" in call_args.kwargs["json"]
```

### When to Use Each Assertion

**Use `assert_called_once()`** when:
- You want to verify the function was called
- You don't care about the arguments
- You want to ensure it wasn't called multiple times

**Use `assert_called_once_with()`** when:
- You know all the arguments
- The arguments are predictable
- You want strict verification

**Use `call_args` inspection** when:
- Some arguments are unpredictable (timestamps, UUIDs)
- You only care about specific arguments
- You need flexible verification

**Use `assert_not_called()`** when:
- Testing conditional logic
- Verifying optimization (caching, short-circuits)
- Ensuring side effects don't happen

### Summary: Verifying Mock Interactions

Mock assertions let you:
- Verify functions were called
- Check call counts
- Inspect arguments
- Verify call order
- Ensure functions weren't called

The key insight: **Mocking isn't just about replacing dependencies—it's about verifying your code uses them correctly.**

In the next section, we'll explore side effects and return values, which let you simulate complex behaviors like exceptions, state changes, and varying responses.

## Mock Side Effects and Return Values

## Mock Side Effects and Return Values

So far, we've configured mocks to return simple values. But real functions do more than return data—they raise exceptions, modify state, and behave differently on subsequent calls. Mock side effects let you simulate all these behaviors.

### Understanding Side Effects

A "side effect" in mocking means "what happens when the mock is called." This can be:
- Returning a value (what we've been doing)
- Raising an exception
- Calling another function
- Returning different values on successive calls

### Iteration 15: Simulating Exceptions

Let's test what happens when the payment API raises a network error:

```python
# payment_processor_with_retry.py
import requests
from datetime import datetime

def process_payment_with_retry(card_number, amount, max_retries=3):
    """Process payment with automatic retry on network errors."""
    for attempt in range(max_retries):
        try:
            response = requests.post(
                "https://api.payment-gateway.com/charge",
                json={
                    "card": card_number,
                    "amount": amount,
                    "currency": "USD",
                    "timestamp": datetime.now().isoformat()
                },
                timeout=30
            )
            
            if response.status_code == 200:
                return {
                    "success": True,
                    "transaction_id": response.json()["transaction_id"],
                    "attempts": attempt + 1
                }
            else:
                return {
                    "success": False,
                    "error": response.json().get("error", "Unknown error"),
                    "attempts": attempt + 1
                }
                
        except requests.exceptions.RequestException as e:
            if attempt == max_retries - 1:
                # Last attempt failed
                return {
                    "success": False,
                    "error": f"Network error after {max_retries} attempts: {str(e)}",
                    "attempts": max_retries
                }
            # Otherwise, retry
            continue
```

Now let's test the retry logic by making the mock raise an exception:

```python
# test_payment_exceptions.py
from unittest.mock import patch, Mock
import requests
from payment_processor_with_retry import process_payment_with_retry

@patch('payment_processor_with_retry.requests.post')
def test_payment_handles_network_error(mock_post):
    """Test that network errors are handled correctly."""
    # Configure mock to raise an exception
    mock_post.side_effect = requests.exceptions.ConnectionError("Network unreachable")
    
    # Call the function
    result = process_payment_with_retry("4111111111111111", 100.00)
    
    # Verify error handling
    assert result["success"] == False
    assert "Network error after 3 attempts" in result["error"]
    assert result["attempts"] == 3
    
    # Verify it tried 3 times
    assert mock_post.call_count == 3
    
    print(f"Result: {result}")
    print(f"Retry attempts: {mock_post.call_count}")
```

Run this test:

```bash
pytest test_payment_exceptions.py::test_payment_handles_network_error -v -s
```

**Output:**
```
test_payment_exceptions.py::test_payment_handles_network_error PASSED
Result: {'success': False, 'error': 'Network error after 3 attempts: Network unreachable', 'attempts': 3}
Retry attempts: 3
```

When `side_effect` is an exception, the mock raises it instead of returning a value. This lets us test error handling without triggering real network failures.

### Iteration 16: Different Responses on Each Call

What if the first two attempts fail but the third succeeds? Use a list of side effects:

```python
@patch('payment_processor_with_retry.requests.post')
def test_payment_succeeds_after_retries(mock_post):
    """Test that payment succeeds after initial failures."""
    # First two calls raise exceptions, third succeeds
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"transaction_id": "txn_12345"}
    
    mock_post.side_effect = [
        requests.exceptions.ConnectionError("Timeout"),
        requests.exceptions.ConnectionError("Timeout"),
        mock_response  # Third attempt succeeds
    ]
    
    result = process_payment_with_retry("4111111111111111", 100.00)
    
    # Verify eventual success
    assert result["success"] == True
    assert result["transaction_id"] == "txn_12345"
    assert result["attempts"] == 3
    
    # Verify it tried 3 times
    assert mock_post.call_count == 3
    
    print(f"Result: {result}")
    print(f"Attempts before success: {result['attempts']}")
```

Run this test:

```bash
pytest test_payment_exceptions.py::test_payment_succeeds_after_retries -v -s
```

**Output:**
```
test_payment_exceptions.py::test_payment_succeeds_after_retries PASSED
Result: {'success': True, 'transaction_id': 'txn_12345', 'attempts': 3}
Attempts before success: 3
```

When `side_effect` is a list, the mock returns/raises each item in sequence. This is perfect for testing retry logic, state machines, or any code that behaves differently on successive calls.

### Iteration 17: Side Effect Functions

For complex behavior, use a function as the side effect:

```python
@patch('payment_processor_with_retry.requests.post')
def test_payment_with_dynamic_behavior(mock_post):
    """Test with side effect that depends on arguments."""
    call_count = 0
    
    def mock_api_call(*args, **kwargs):
        """Simulate API that fails for large amounts."""
        nonlocal call_count
        call_count += 1
        
        amount = kwargs["json"]["amount"]
        
        mock_response = Mock()
        if amount > 1000:
            # Large amounts fail
            mock_response.status_code = 400
            mock_response.json.return_value = {"error": "Amount too large"}
        else:
            # Small amounts succeed
            mock_response.status_code = 200
            mock_response.json.return_value = {"transaction_id": f"txn_{call_count}"}
        
        return mock_response
    
    mock_post.side_effect = mock_api_call
    
    # Test small amount (should succeed)
    result1 = process_payment_with_retry("4111111111111111", 100.00)
    assert result1["success"] == True
    
    # Test large amount (should fail)
    result2 = process_payment_with_retry("4111111111111111", 2000.00)
    assert result2["success"] == False
    assert result2["error"] == "Amount too large"
    
    print(f"Small amount result: {result1}")
    print(f"Large amount result: {result2}")
```

Run this test:

```bash
pytest test_payment_exceptions.py::test_payment_with_dynamic_behavior -v -s
```

**Output:**
```
test_payment_exceptions.py::test_payment_with_dynamic_behavior PASSED
Small amount result: {'success': True, 'transaction_id': 'txn_1', 'attempts': 1}
Large amount result: {'success': False, 'error': 'Amount too large', 'attempts': 1}
```

When `side_effect` is a function, the mock calls it with the same arguments it received. This lets you implement arbitrarily complex mock behavior.

### Iteration 18: Combining return_value and side_effect

You can't use both `return_value` and `side_effect` on the same mock—`side_effect` takes precedence. But you can use `side_effect` to return values:

```python
def test_side_effect_vs_return_value():
    """Demonstrate side_effect precedence."""
    mock_func = Mock()
    
    # Set both return_value and side_effect
    mock_func.return_value = "from return_value"
    mock_func.side_effect = ["from side_effect"]
    
    # side_effect wins
    result = mock_func()
    assert result == "from side_effect"
    
    print(f"Result: {result}")
```

**Rule**: If `side_effect` is set, `return_value` is ignored.

### Iteration 19: Resetting Mocks

Sometimes you need to reset a mock's state between test sections:

```python
def test_resetting_mocks():
    """Demonstrate mock reset methods."""
    mock_func = Mock(return_value="initial")
    
    # Call it a few times
    mock_func("arg1")
    mock_func("arg2")
    assert mock_func.call_count == 2
    
    # Reset call history but keep configuration
    mock_func.reset_mock()
    assert mock_func.call_count == 0
    assert mock_func.return_value == "initial"  # Configuration preserved
    
    # Call again
    result = mock_func("arg3")
    assert result == "initial"
    assert mock_func.call_count == 1
    
    print(f"After reset, call count: {mock_func.call_count}")
    print(f"Return value preserved: {mock_func.return_value}")
```

`reset_mock()` clears:
- `called`
- `call_count`
- `call_args`
- `call_args_list`

But preserves:
- `return_value`
- `side_effect`
- Attribute configurations

### Iteration 20: Mocking Properties

Sometimes you need to mock object properties, not just methods:

```python
# user_service.py
class UserService:
    def __init__(self, api_client):
        self.api_client = api_client
    
    def get_user_status(self, user_id):
        """Get user status from API."""
        if not self.api_client.is_connected:
            return {"error": "Not connected"}
        
        response = self.api_client.get(f"/users/{user_id}")
        return response.json()
```

```python
# test_user_service.py
from unittest.mock import Mock, PropertyMock
from user_service import UserService

def test_user_service_checks_connection():
    """Test that service checks connection status."""
    mock_client = Mock()
    
    # Mock the is_connected property
    type(mock_client).is_connected = PropertyMock(return_value=False)
    
    service = UserService(mock_client)
    result = service.get_user_status(123)
    
    assert result == {"error": "Not connected"}
    
    # Verify the property was accessed
    type(mock_client).is_connected.assert_called()
    
    print(f"Result: {result}")
```

`PropertyMock` is used to mock properties. The syntax is unusual because properties are defined on the class, not the instance.

### Side Effect Patterns Summary

| Pattern | Use Case | Example |
|---------|----------|---------|
| Single exception | Test error handling | `side_effect=ValueError("error")` |
| List of values | Test retry logic | `side_effect=[error, error, success]` |
| Function | Complex behavior | `side_effect=lambda x: x * 2` |
| Iterator | Infinite sequence | `side_effect=itertools.count()` |

### Common Side Effect Mistakes

#### Mistake 1: Forgetting to Raise Exceptions

**Wrong**:
```python
mock_func.side_effect = ValueError("error")  # Returns the exception object
result = my_function()  # Gets ValueError instance, doesn't raise
```

**Right**:
```python
mock_func.side_effect = ValueError("error")  # Raises the exception
# OR
mock_func.side_effect = lambda: ValueError("error")  # Also raises
```

#### Mistake 2: List Too Short

**Wrong**:
```python
mock_func.side_effect = ["first", "second"]
mock_func()  # Returns "first"
mock_func()  # Returns "second"
mock_func()  # Raises StopIteration!
```

**Right**:
```python
# Make sure list is long enough
mock_func.side_effect = ["first", "second", "third"]
# OR use a function for infinite behavior
mock_func.side_effect = lambda: "always this"
```

#### Mistake 3: Mixing return_value and side_effect

**Wrong**:
```python
mock_func.return_value = "value"
mock_func.side_effect = ["other"]
result = mock_func()  # Returns "other", not "value"
```

**Right**:
```python
# Choose one approach
mock_func.side_effect = ["value"]  # Use side_effect
# OR
mock_func.return_value = "value"  # Use return_value
```

### When to Use Side Effects

**Use exceptions** when:
- Testing error handling
- Simulating network failures
- Testing retry logic

**Use lists** when:
- Testing state changes
- Simulating retry scenarios
- Testing code that polls or iterates

**Use functions** when:
- Behavior depends on arguments
- You need complex logic
- You want to track state across calls

**Use return_value** when:
- Behavior is simple and constant
- You don't need exceptions or state changes

### Summary: The Power of Side Effects

Side effects let you:
- Simulate exceptions and errors
- Return different values on successive calls
- Implement complex, stateful behavior
- Test retry logic and error handling
- Mock properties and attributes

The key insight: **Mocks can simulate any behavior, not just simple return values.**

In the final section, we'll combine mocks with pytest fixtures to create reusable, maintainable test infrastructure.

## Combining Mocks and Fixtures

## Combining Mocks and Fixtures

We've learned how to create mocks and how to use fixtures (Chapter 4). Now we'll combine them to create reusable, maintainable test infrastructure. This is where mocking becomes truly powerful in real-world projects.

### The Problem: Repetitive Mock Setup

Look at our tests so far. Every test that mocks `requests.post` repeats the same setup:

```python
@patch('payment_processor.requests.post')
def test_something(mock_post):
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"transaction_id": "txn_12345"}
    mock_post.return_value = mock_response
    # ... actual test code
```

This setup appears in every test. If we need to change the mock response structure, we have to update every test. This violates the DRY (Don't Repeat Yourself) principle.

### Iteration 21: Mock Fixtures

Let's create a fixture that provides a pre-configured mock:

```python
# conftest.py
import pytest
from unittest.mock import Mock, patch

@pytest.fixture
def mock_payment_api():
    """Fixture that mocks the payment API with success response."""
    with patch('payment_processor.requests.post') as mock_post:
        # Configure default success response
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {"transaction_id": "txn_12345"}
        mock_post.return_value = mock_response
        
        yield mock_post
```

Now tests can use this fixture instead of repeating the setup:

```python
# test_payment_with_fixtures.py
from payment_processor import process_payment

def test_successful_payment(mock_payment_api):
    """Test using the mock fixture."""
    result = process_payment("4111111111111111", 100.00)
    
    assert result["success"] == True
    assert result["transaction_id"] == "txn_12345"
    
    # The fixture provides the mock
    mock_payment_api.assert_called_once()
```

Run this test:

```bash
pytest test_payment_with_fixtures.py::test_successful_payment -v
```

**Output:**
```
test_payment_with_fixtures.py::test_successful_payment PASSED
```

The fixture handles all the mock setup and teardown. Tests are now cleaner and more focused on behavior.

### Iteration 22: Parameterized Mock Fixtures

What if different tests need different mock responses? Make the fixture configurable:

```python
# conftest.py (updated)
@pytest.fixture
def mock_payment_api_factory():
    """Fixture factory that creates configured payment API mocks."""
    def _create_mock(status_code=200, transaction_id="txn_12345", error=None):
        with patch('payment_processor.requests.post') as mock_post:
            mock_response = Mock()
            mock_response.status_code = status_code
            
            if status_code == 200:
                mock_response.json.return_value = {"transaction_id": transaction_id}
            else:
                mock_response.json.return_value = {"error": error or "Unknown error"}
            
            mock_post.return_value = mock_response
            return mock_post
    
    return _create_mock
```

Now tests can customize the mock behavior:

```python
def test_successful_payment_with_factory(mock_payment_api_factory):
    """Test success scenario with factory."""
    mock_post = mock_payment_api_factory(status_code=200, transaction_id="txn_custom")
    
    with mock_post:
        result = process_payment("4111111111111111", 100.00)
        assert result["transaction_id"] == "txn_custom"

def test_failed_payment_with_factory(mock_payment_api_factory):
    """Test failure scenario with factory."""
    mock_post = mock_payment_api_factory(status_code=400, error="Invalid card")
    
    with mock_post:
        result = process_payment("0000000000000000", 100.00)
        assert result["error"] == "Invalid card"
```

This pattern is called a "fixture factory"—a fixture that returns a function for creating configured objects.

### Iteration 23: Fixtures with Automatic Patching

For even cleaner tests, make the fixture automatically apply the patch:

```python
# conftest.py (updated)
@pytest.fixture
def payment_api_mock():
    """Fixture that automatically patches and configures payment API."""
    with patch('payment_processor.requests.post') as mock_post:
        # Default configuration
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {"transaction_id": "txn_12345"}
        mock_post.return_value = mock_response
        
        # Yield the mock for test customization
        yield mock_post
        
        # Automatic cleanup happens here
```

```python
def test_with_auto_patching(payment_api_mock):
    """Test with automatic patching fixture."""
    # The patch is already active
    result = process_payment("4111111111111111", 100.00)
    
    assert result["success"] == True
    payment_api_mock.assert_called_once()

def test_with_custom_response(payment_api_mock):
    """Test with customized mock response."""
    # Reconfigure the mock for this test
    mock_response = Mock()
    mock_response.status_code = 400
    mock_response.json.return_value = {"error": "Declined"}
    payment_api_mock.return_value = mock_response
    
    result = process_payment("4111111111111111", 100.00)
    
    assert result["success"] == False
    assert result["error"] == "Declined"
```

### Iteration 24: Fixtures for Complex Mock Scenarios

For complex systems, create fixtures that mock entire subsystems:

```python
# payment_system.py
import requests
from datetime import datetime

class PaymentGateway:
    """Payment gateway client."""
    
    def __init__(self, api_key):
        self.api_key = api_key
        self.base_url = "https://api.payment-gateway.com"
    
    def charge(self, card_number, amount, currency="USD"):
        """Charge a card."""
        response = requests.post(
            f"{self.base_url}/charge",
            headers={"Authorization": f"Bearer {self.api_key}"},
            json={
                "card": card_number,
                "amount": amount,
                "currency": currency,
                "timestamp": datetime.now().isoformat()
            },
            timeout=30
        )
        return response

class PaymentProcessor:
    """High-level payment processor."""
    
    def __init__(self, gateway):
        self.gateway = gateway
    
    def process_payment(self, card_number, amount, currency="USD"):
        """Process a payment."""
        response = self.gateway.charge(card_number, amount, currency)
        
        if response.status_code == 200:
            data = response.json()
            return {
                "success": True,
                "transaction_id": data["transaction_id"]
            }
        else:
            return {
                "success": False,
                "error": response.json().get("error", "Unknown error")
            }
```

Create fixtures for each component:

```python
# conftest.py (updated)
@pytest.fixture
def mock_gateway_response():
    """Fixture providing a mock gateway response."""
    mock_response = Mock()
    mock_response.status_code = 200
    mock_response.json.return_value = {"transaction_id": "txn_12345"}
    return mock_response

@pytest.fixture
def mock_gateway(mock_gateway_response):
    """Fixture providing a mock payment gateway."""
    gateway = Mock(spec=PaymentGateway)
    gateway.charge.return_value = mock_gateway_response
    return gateway

@pytest.fixture
def payment_processor(mock_gateway):
    """Fixture providing a payment processor with mocked gateway."""
    return PaymentProcessor(mock_gateway)
```

Now tests can use high-level fixtures:

```python
# test_payment_system.py
def test_payment_processor_success(payment_processor, mock_gateway):
    """Test payment processor with mocked gateway."""
    result = payment_processor.process_payment("4111111111111111", 100.00)
    
    assert result["success"] == True
    assert result["transaction_id"] == "txn_12345"
    
    # Verify the gateway was called correctly
    mock_gateway.charge.assert_called_once_with(
        "4111111111111111", 100.00, currency="USD"
    )

def test_payment_processor_failure(payment_processor, mock_gateway, mock_gateway_response):
    """Test payment processor with failure response."""
    # Reconfigure the response for this test
    mock_gateway_response.status_code = 400
    mock_gateway_response.json.return_value = {"error": "Declined"}
    
    result = payment_processor.process_payment("4111111111111111", 100.00)
    
    assert result["success"] == False
    assert result["error"] == "Declined"
```

### Fixture Composition Benefits

This fixture composition provides:

1. **Reusability**: Mock setup is defined once, used everywhere
2. **Maintainability**: Change mock behavior in one place
3. **Clarity**: Tests focus on behavior, not mock setup
4. **Flexibility**: Tests can customize fixtures as needed
5. **Hierarchy**: Complex fixtures build on simpler ones

### Iteration 25: Fixture Scopes with Mocks

Be careful with fixture scopes when using mocks. Mocks should usually be function-scoped to avoid test pollution:

```python
# conftest.py (updated)
@pytest.fixture(scope="function")  # Explicit function scope
def mock_payment_api():
    """Function-scoped mock (default)."""
    with patch('payment_processor.requests.post') as mock_post:
        mock_response = Mock()
        mock_response.status_code = 200
        mock_response.json.return_value = {"transaction_id": "txn_12345"}
        mock_post.return_value = mock_response
        yield mock_post

@pytest.fixture(scope="module")  # Module scope - be careful!
def shared_mock_config():
    """Module-scoped configuration (not the mock itself)."""
    return {
        "default_status": 200,
        "default_transaction_id": "txn_12345"
    }
```

**Rule**: Mock fixtures should almost always be function-scoped. Configuration fixtures can be module or session-scoped.

### Iteration 26: Combining Fixtures with Parametrize

You can combine mock fixtures with parametrized tests:

```python
import pytest

@pytest.fixture
def mock_payment_api_with_status(request):
    """Fixture that uses parametrized status code."""
    status_code = request.param
    
    with patch('payment_processor.requests.post') as mock_post:
        mock_response = Mock()
        mock_response.status_code = status_code
        
        if status_code == 200:
            mock_response.json.return_value = {"transaction_id": "txn_12345"}
        else:
            mock_response.json.return_value = {"error": "Error"}
        
        mock_post.return_value = mock_response
        yield mock_post

@pytest.mark.parametrize('mock_payment_api_with_status', [200, 400, 500], indirect=True)
def test_payment_with_various_statuses(mock_payment_api_with_status):
    """Test payment with different status codes."""
    result = process_payment("4111111111111111", 100.00)
    
    status = mock_payment_api_with_status.return_value.status_code
    if status == 200:
        assert result["success"] == True
    else:
        assert result["success"] == False
    
    print(f"Status {status}: {result}")
```

The `indirect=True` parameter tells pytest to pass the parameter to the fixture instead of directly to the test.

### Best Practices for Mock Fixtures

#### 1. Keep Fixtures Focused

**Bad**:
```python
@pytest.fixture
def everything_mocked():
    """Mocks everything at once."""
    with patch('module.api') as api, \
         patch('module.db') as db, \
         patch('module.cache') as cache:
        # Configure everything...
        yield api, db, cache
```

**Good**:
```python
@pytest.fixture
def mock_api():
    """Mocks only the API."""
    with patch('module.api') as api:
        yield api

@pytest.fixture
def mock_db():
    """Mocks only the database."""
    with patch('module.db') as db:
        yield db
```

#### 2. Provide Sensible Defaults

**Bad**:
```python
@pytest.fixture
def mock_api():
    """Unconfigured mock."""
    with patch('module.api') as api:
        yield api  # Tests must configure everything
```

**Good**:
```python
@pytest.fixture
def mock_api():
    """Pre-configured mock with sensible defaults."""
    with patch('module.api') as api:
        api.get.return_value = {"status": "ok"}
        yield api  # Tests can override if needed
```

#### 3. Use Fixture Factories for Variation

**Bad**:
```python
@pytest.fixture
def mock_api_success():
    # ...

@pytest.fixture
def mock_api_failure():
    # ...

@pytest.fixture
def mock_api_timeout():
    # ...
```

**Good**:
```python
@pytest.fixture
def mock_api_factory():
    """Factory for creating API mocks."""
    def _create(status="success"):
        # Configure based on status
        pass
    return _create
```

#### 4. Document Fixture Behavior

**Bad**:
```python
@pytest.fixture
def mock_api():
    with patch('module.api') as api:
        api.get.return_value = {"data": [1, 2, 3]}
        yield api
```

**Good**:
```python
@pytest.fixture
def mock_api():
    """Mock API client with default success response.
    
    Returns:
        Mock: Configured mock with get() returning {"data": [1, 2, 3]}
        
    Example:
        def test_something(mock_api):
            # Customize if needed
            mock_api.get.return_value = {"data": [4, 5, 6]}
    """
    with patch('module.api') as api:
        api.get.return_value = {"data": [1, 2, 3]}
        yield api
```

### Summary: The Complete Journey

Let's review our progression through this chapter:

| Iteration | Problem | Solution | Technique |
|-----------|---------|----------|-----------|
| 0 | Tests make real network calls | Need to replace dependencies | Mocking concept |
| 1 | Manual dependency injection | Inject mocks | `Mock()` objects |
| 2 | Direct replacement is verbose | Automate replacement | Manual patching |
| 3 | Manual cleanup is error-prone | Automatic cleanup | `@patch` decorator |
| 4 | Need to test errors | Simulate failures | Mock configuration |
| 5 | Multiple dependencies | Stack patches | Multiple `@patch` |
| 6 | Partial test mocking | Context manager | `with patch()` |
| 7 | Configuration values | Patch attributes | Attribute patching |
| 8 | Verify function was called | Check interactions | `assert_called()` |
| 9 | Verify call count | Count calls | `assert_called_once()` |
| 10 | Verify arguments | Inspect arguments | `assert_called_with()` |
| 11 | Unpredictable arguments | Partial verification | `call_args` inspection |
| 12 | Multiple calls | Inspect all calls | `call_args_list` |
| 13 | Verify no calls | Check absence | `assert_not_called()` |
| 14 | Verify specific call | Find in history | `assert_any_call()` |
| 15 | Test error handling | Raise exceptions | `side_effect` with exception |
| 16 | Test retry logic | Different responses | `side_effect` with list |
| 17 | Complex behavior | Dynamic responses | `side_effect` with function |
| 18 | Mock properties | Property mocking | `PropertyMock` |
| 19 | Reset between tests | Clear state | `reset_mock()` |
| 20 | Repetitive setup | Reusable mocks | Fixtures |
| 21 | Different scenarios | Configurable mocks | Fixture factories |
| 22 | Automatic patching | Integrated fixtures | Fixture with `patch()` |
| 23 | Complex systems | Layered mocks | Fixture composition |
| 24 | Scope management | Proper isolation | Function-scoped fixtures |
| 25 | Parametrized mocks | Indirect parameters | `indirect=True` |

### Final Implementation: Production-Ready Test Suite

Here's a complete, production-ready test suite using all the techniques we've learned:

```python
# conftest.py - Complete fixture setup
import pytest
from unittest.mock import Mock, patch

@pytest.fixture
def mock_payment_response():
    """Fixture providing a configurable mock response."""
    def _create_response(status_code=200, transaction_id="txn_12345", error=None):
        mock_response = Mock()
        mock_response.status_code = status_code
        
        if status_code == 200:
            mock_response.json.return_value = {"transaction_id": transaction_id}
        else:
            mock_response.json.return_value = {"error": error or "Unknown error"}
        
        return mock_response
    
    return _create_response

@pytest.fixture
def mock_payment_api(mock_payment_response):
    """Fixture providing a mocked payment API with sensible defaults."""
    with patch('payment_processor.requests.post') as mock_post:
        # Default: successful response
        mock_post.return_value = mock_payment_response()
        yield mock_post

@pytest.fixture
def mock_payment_gateway(mock_payment_response):
    """Fixture providing a mocked PaymentGateway."""
    gateway = Mock(spec=PaymentGateway)
    gateway.charge.return_value = mock_payment_response()
    return gateway

@pytest.fixture
def payment_processor(mock_payment_gateway):
    """Fixture providing a PaymentProcessor with mocked gateway."""
    return PaymentProcessor(mock_payment_gateway)
```

```python
# test_payment_complete.py - Complete test suite
import pytest
from payment_processor import process_payment
from payment_system import PaymentProcessor

class TestPaymentProcessor:
    """Test suite for payment processing."""
    
    def test_successful_payment(self, mock_payment_api):
        """Test successful payment processing."""
        result = process_payment("4111111111111111", 100.00)
        
        assert result["success"] == True
        assert result["transaction_id"] == "txn_12345"
        mock_payment_api.assert_called_once()
    
    def test_failed_payment(self, mock_payment_api, mock_payment_response):
        """Test failed payment processing."""
        mock_payment_api.return_value = mock_payment_response(
            status_code=400,
            error="Invalid card"
        )
        
        result = process_payment("0000000000000000", 100.00)
        
        assert result["success"] == False
        assert result["error"] == "Invalid card"
    
    def test_network_error_handling(self, mock_payment_api):
        """Test network error handling."""
        import requests
        mock_payment_api.side_effect = requests.exceptions.ConnectionError("Network error")
        
        with pytest.raises(requests.exceptions.ConnectionError):
            process_payment("4111111111111111", 100.00)
    
    @pytest.mark.parametrize("amount,currency", [
        (100.00, "USD"),
        (200.00, "EUR"),
        (300.00, "GBP"),
    ])
    def test_various_amounts_and_currencies(self, mock_payment_api, amount, currency):
        """Test payment with various amounts and currencies."""
        result = process_payment("4111111111111111", amount, currency=currency)
        
        assert result["success"] == True
        
        # Verify correct arguments were passed
        call_args = mock_payment_api.call_args
        assert call_args.kwargs["json"]["amount"] == amount
        assert call_args.kwargs["json"]["currency"] == currency

class TestPaymentSystem:
    """Test suite for the complete payment system."""
    
    def test_processor_uses_gateway(self, payment_processor, mock_payment_gateway):
        """Test that processor correctly uses gateway."""
        result = payment_processor.process_payment("4111111111111111", 100.00)
        
        assert result["success"] == True
        mock_payment_gateway.charge.assert_called_once_with(
            "4111111111111111", 100.00, currency="USD"
        )
    
    def test_processor_handles_gateway_errors(
        self, payment_processor, mock_payment_gateway, mock_payment_response
    ):
        """Test that processor handles gateway errors."""
        mock_payment_gateway.charge.return_value = mock_payment_response(
            status_code=500,
            error="Gateway error"
        )
        
        result = payment_processor.process_payment("4111111111111111", 100.00)
        
        assert result["success"] == False
        assert result["error"] == "Gateway error"
```

### Decision Framework: When to Use Each Approach

| Scenario | Recommended Approach | Why |
|----------|---------------------|-----|
| Simple, one-off mock | `@patch` decorator | Clean, declarative |
| Reusable mock setup | Fixture | DRY principle |
| Multiple test variations | Fixture factory | Flexibility |
| Complex system | Fixture composition | Maintainability |
| Partial test mocking | Context manager | Fine-grained control |
| Parametrized scenarios | Indirect parametrization | Combines benefits |

### Lessons Learned

**Mocking is about isolation**: Test your code, not external systems.

**Mocking is about control**: Simulate any scenario, including errors.

**Mocking is about verification**: Ensure your code uses dependencies correctly.

**Fixtures make mocking maintainable**: Reusable mock setup reduces duplication.

**Composition creates flexibility**: Build complex fixtures from simple ones.

**Documentation prevents confusion**: Clear docstrings explain fixture behavior.

The journey from manual mocking to fixture-based test infrastructure represents the evolution from beginner to professional testing practices. Master these patterns, and you'll write tests that are fast, reliable, and maintainable.
