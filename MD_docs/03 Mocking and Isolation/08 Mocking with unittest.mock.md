# Chapter 8: Mocking with unittest.mock

## Why Mock?

## Why Mock?

So far, we've tested functions that are self-contained. They take inputs, perform calculations, and return outputs. But what happens when your code isn't self-contained? What happens when it depends on something outside of its control?

Consider a function that gets the current weather for a city by calling an external API.

```python
# src/weather_reporter.py
import requests

def get_weather_data(city: str) -> dict:
    """Fetches weather data from a (fictional) live API."""
    try:
        response = requests.get(f"https://api.weather.com/data?city={city}")
        response.raise_for_status()  # Raise an exception for bad status codes
        return response.json()
    except requests.exceptions.RequestException as e:
        print(f"API request failed: {e}")
        return {"error": str(e)}

def format_weather_report(city: str) -> str:
    """Formats a human-readable weather report."""
    data = get_weather_data(city)
    if "error" in data:
        return f"Sorry, could not retrieve weather for {city}."
    
    temp = data.get("temperature")
    condition = data.get("condition")
    
    if temp is None or condition is None:
        return f"Incomplete weather data for {city}."
        
    return f"The weather in {city} is {temp}°C and {condition}."
```

Now, let's think about how we would test `format_weather_report()`. If we call it directly in a test, we run into several major problems:

1.  **Unreliability:** The external weather API could be down. If it is, our test will fail, even if our `format_weather_report` function is perfectly correct. A failing test should indicate a bug in *our* code, not someone else's.
2.  **Slowness:** Network requests are slow. A test suite with hundreds of tests making real network calls would take minutes or even hours to run, discouraging developers from running it frequently.
3.  **Cost & Rate Limiting:** Many APIs are paid services or have rate limits. Running your test suite hundreds of times a day could incur costs or get your IP address blocked.
4.  **Unpredictability:** The weather in London changes! One day the API might return "Sunny," the next "Rainy." We can't write a stable assertion like `assert report == "The weather in London is 15°C and Sunny."` because the data is always changing. We also can't easily test edge cases, like what happens if the API returns incomplete data or an error.

This is the core problem that **mocking** solves.

Mocking allows us to **isolate the code under test** from its external dependencies. Instead of calling the real `requests.get`, we'll replace it with a "stunt double"—a fake version that is completely under our control. This lets us test our `format_weather_report` logic in isolation, ensuring it works correctly regardless of what the outside world is doing.

## What Is a Mock?

## What Is a Mock?

A **mock object** is a simulated object that mimics the behavior of a real object in a controlled way. Think of it as a stunt double in a movie. When a scene is too dangerous or requires a specific skill the main actor doesn't have, a stunt double steps in. They look and act enough like the real actor for the scene to work, but they are completely controlled by the film's director.

In testing, our test function is the director. The mock object is our stunt double for a real, problematic dependency (like the `requests` library).

A mock object, at its core, does two things:

1.  **Simulates Behavior:** You can tell a mock object how to behave. You can configure its methods to return specific values, or even to raise exceptions, allowing you to simulate any possible scenario from the real dependency. For our weather example, we can tell our mock `requests.get` to return a specific JSON payload representing "Sunny" or an error code representing "API is down."
2.  **Records Interaction:** A mock object spies on the code that uses it. It keeps a detailed log of every time its methods or attributes are accessed. After our code under test has run, we can inspect the mock to ask questions like:
    *   "Were you called?"
    *   "How many times were you called?"
    *   "What arguments were you called with?"

This allows us to verify that our code is interacting with its dependencies correctly.

Python's standard library includes a powerful mocking library called `unittest.mock`. While its name comes from the `unittest` framework, it is a standalone library that works perfectly with pytest and is the industry standard for mocking in Python. We'll be using it extensively.

### Test Doubles: A Family of Fakes

"Mock" is often used as a catch-all term, but it's technically part of a larger family of objects called **Test Doubles**. You might hear these terms:

*   **Dummy:** An object that is passed around but never actually used. Usually just to fill a parameter list.
*   **Stub:** An object that provides canned answers to calls made during the test. It doesn't record interactions.
*   **Spy:** An object that records information on how it was called. It's a "stub with recording capabilities."
*   **Fake:** An object with a working implementation, but it's not the real production one. For example, an in-memory database instead of a real PostgreSQL database.
*   **Mock:** An object that is pre-programmed with expectations which form a specification of the calls they are expected to receive. It can throw an exception if it receives a call it doesn't expect.

`unittest.mock` provides objects that can act as stubs, spies, and mocks. For simplicity, we'll mostly use the term "mock," as it's the most common in the Python community. The key takeaway is that we are replacing a real component with a controllable fake for the purpose of a test.

## Creating Mocks with Mock()

## Creating Mocks with Mock()

Let's start by creating a mock object directly to see how it behaves. The primary class for this is `Mock` from the `unittest.mock` library.

```python
# test_mock_basics.py
from unittest.mock import Mock

def test_basic_mock_behavior():
    # Create a mock object
    mock_api_client = Mock()

    # You can access any attribute on it, and it will return another Mock!
    print(f"Attribute access: {mock_api_client.get_user}")
    # Output: Attribute access: <Mock name='mock.get_user' id='...'>

    # You can call any method on it, and it will also return another Mock!
    print(f"Method call: {mock_api_client.get_user(id=1)}")
    # Output: Method call: <Mock name='mock.get_user()' id='...'>

    # This is useful, but not what we usually want.
    # We want to control the return value.
    mock_api_client.get_user.return_value = {"name": "Alice", "id": 1}

    # Now when we call it, we get our specified value
    user_data = mock_api_client.get_user(id=1)
    assert user_data == {"name": "Alice", "id": 1}
```

The `Mock` object is incredibly flexible. It allows any attribute access or method call by default, creating new `Mock` objects on the fly. This is powerful but can sometimes hide typos.

The real utility comes from configuring its behavior and then asserting how it was used.

### Configuring Return Values and Attributes

You can set the `return_value` of a mock method or set attributes directly.

```python
# test_mock_basics.py
from unittest.mock import Mock

def test_configuring_a_mock():
    # Create a mock for a database connection object
    mock_db_conn = Mock()

    # Configure an attribute
    mock_db_conn.is_connected = True

    # Configure a method's return value
    mock_db_conn.get_user_by_id.return_value = {
        "username": "testuser",
        "email": "test@example.com"
    }

    # Now we can use it as if it were a real object
    assert mock_db_conn.is_connected is True
    
    user = mock_db_conn.get_user_by_id(123)
    assert user["username"] == "testuser"
```

This is the foundation of mocking: creating a fake object and telling it how to respond when our code interacts with it. However, creating a mock like this doesn't help us test our `format_weather_report` function. That function doesn't accept a `requests` object as an argument; it imports and uses it directly.

To solve this, we need to temporarily replace the *real* `requests` object in our application's namespace with our *mock* object. This process is called **patching**.

## Patching Functions with @patch

## Patching Functions with @patch

Patching is the act of intercepting calls to functions or objects and replacing them with something else, usually a mock. The `unittest.mock.patch` function is our primary tool for this, and it's most commonly used as a decorator.

Let's finally write a proper, isolated test for our `format_weather_report` function.

```python
# tests/test_weather_reporter.py
from unittest.mock import patch
from src.weather_reporter import format_weather_report

# The target string is 'path.to.module.object_to_patch'
# We are patching 'requests.get' inside the 'weather_reporter' module.
@patch("src.weather_reporter.requests.get")
def test_format_weather_report_sunny(mock_requests_get):
    """
    Test the report formatter with a successful, sunny weather API response.
    """
    # Configure the mock to behave like a successful API call
    mock_response = mock_requests_get.return_value
    mock_response.status_code = 200
    mock_response.json.return_value = {
        "temperature": 25,
        "condition": "Sunny"
    }

    # Call the function we are testing
    report = format_weather_report("London")

    # Assert that our function formatted the report correctly
    assert report == "The weather in London is 25°C and Sunny."
```

Let's break this down, because it's the most important concept in this chapter.

### The `@patch` Decorator

`@patch("src.weather_reporter.requests.get")`

This decorator does all the magic. During the execution of `test_format_weather_report_sunny`, it finds the `get` object inside the `requests` module *as it is seen by `src.weather_reporter`* and replaces it with a mock object.

After the test function finishes (whether it passes, fails, or raises an error), `patch` automatically restores the original `requests.get`, so other tests are not affected.

### The Mock Argument

`def test_format_weather_report_sunny(mock_requests_get):`

The `patch` decorator creates a mock object and passes it into our test function as an argument. The name of the argument (`mock_requests_get`) can be anything you like, but it's good practice to name it descriptively. This is the object we use to configure behavior and make assertions.

### The Target String: The Most Common Point of Confusion

`"src.weather_reporter.requests.get"`

Why this specific string? This is critical. **You must patch the object where it is looked up, not where it is defined.**

Our code under test is in `src/weather_reporter.py`. Inside that file, the line `response = requests.get(...)` looks up the name `get` within the `requests` module that was imported into that file's namespace. Therefore, we must patch `requests.get` *within the `src.weather_reporter` module*.

A common mistake is to try patching `'requests.get'`. This would patch the `get` function in the original `requests` library, but our `weather_reporter` module has already imported its own reference to it. The patch would have no effect on our code under test.

**Think of it like this:** You need to change the tool in the toolbox that your code is actually using.

### Testing an Error Case

Now, let's see how easy it is to test the "API is down" scenario.

```python
# tests/test_weather_reporter.py
from unittest.mock import patch, Mock
from requests.exceptions import RequestException
from src.weather_reporter import format_weather_report

# ... (previous test) ...

@patch("src.weather_reporter.requests.get")
def test_format_weather_report_api_error(mock_requests_get):
    """
    Test the report formatter when the API call fails.
    """
    # Configure the mock to raise an exception, just like the real library would
    mock_requests_get.side_effect = RequestException("API is down")

    # Call the function we are testing
    report = format_weather_report("Paris")

    # Assert that our function handles the error gracefully
    assert report == "Sorry, could not retrieve weather for Paris."
```

Without mocking, this test would be nearly impossible to write reliably. With mocking, it's trivial. We simply configure our mock to raise an exception when called, and then verify that our error-handling logic works as expected. We'll cover `side_effect` in more detail shortly.

## Common Mock Assertions (assert_called, assert_called_with)

## Common Mock Assertions (assert_called, assert_called_with)

The first part of mocking is controlling behavior. The second, equally important part is verifying that our code interacted with the dependency as we expected. Mock objects provide a suite of assertion methods for this purpose.

Let's go back to our successful weather test and add some assertions about how the mock was used.

```python
# tests/test_weather_reporter.py
from unittest.mock import patch
from src.weather_reporter import format_weather_report

@patch("src.weather_reporter.requests.get")
def test_format_weather_report_sunny_with_call_assertions(mock_requests_get):
    """
    Test the report formatter and verify the API call.
    """
    mock_response = mock_requests_get.return_value
    mock_response.status_code = 200
    mock_response.json.return_value = {
        "temperature": 25,
        "condition": "Sunny"
    }

    # Call the function we are testing
    report = format_weather_report("London")

    # Assert the output is correct
    assert report == "The weather in London is 25°C and Sunny."

    # --- New Assertions ---
    # Verify that requests.get was called
    mock_requests_get.assert_called()

    # Verify it was called exactly once
    mock_requests_get.assert_called_once()

    # Verify it was called with the correct arguments
    mock_requests_get.assert_called_once_with("https://api.weather.com/data?city=London")
```

These assertions allow us to confirm that our function is constructing the API URL correctly.

### Reading Failure Messages

Pytest's integration with `unittest.mock` provides incredibly detailed failure messages. Let's say we made a typo in our code and called the API with `city=london` (lowercase).

```python
# In src/weather_reporter.py (with a bug)
def get_weather_data(city: str) -> dict:
    # ...
    # BUG: city is not capitalized as expected
    response = requests.get(f"https://api.weather.com/data?city={city.lower()}") 
    # ...
```

Our `assert_called_once_with` would fail with a very helpful message:

```bash
>       mock_requests_get.assert_called_once_with("https://api.weather.com/data?city=London")
E       AssertionError: expected call not found.
E       Expected: get('https://api.weather.com/data?city=London')
E       Actual: get('https://api.weather.com/data?city=london')
```

This is an example of **treating errors as data**. The output isn't just "failure"; it's a map showing you exactly what went wrong.

### Other Useful Assertions

Here are some other common assertion methods:

*   `mock_object.assert_not_called()`: Verifies the mock was never called. Useful for testing logic branches where a dependency *shouldn't* be touched.
*   `mock_object.assert_any_call(*args, **kwargs)`: Verifies the mock was called with the given arguments at least once, even if it was called with other arguments as well.
*   `mock_object.call_count`: An integer property that tells you how many times the mock was called. You can assert against it directly: `assert mock_object.call_count == 2`.

Let's write a test for the `assert_not_called` case. Imagine we have a function that caches results and should only call the API if the city is not in the cache.

```python
# src/cached_weather.py
import requests

CACHE = {}

def get_weather_with_cache(city: str) -> dict:
    if city in CACHE:
        return CACHE[city]
    
    # This part should only be reached if the city is not in the cache
    response = requests.get(f"https://api.weather.com/data?city={city}")
    response.raise_for_status()
    data = response.json()
    CACHE[city] = data
    return data
```

```python
# tests/test_cached_weather.py
from unittest.mock import patch
from src.cached_weather import get_weather_with_cache, CACHE

@patch("src.cached_weather.requests.get")
def test_get_weather_from_cache(mock_requests_get):
    # Pre-populate the cache
    CACHE["Tokyo"] = {"temperature": 18, "condition": "Cloudy"}

    # Call the function for a cached city
    get_weather_with_cache("Tokyo")

    # Verify that the real API was NOT called
    mock_requests_get.assert_not_called()

    # Clean up the cache for other tests
    CACHE.clear()
```

## Mock Side Effects and Return Values

## Mock Side Effects and Return Values

We've already seen `return_value` and a brief example of `side_effect`. Let's explore these two powerful configuration options in more detail. They are the primary ways you control how your mock behaves when called.

### `return_value`: The Simple Case

`return_value` is an attribute on a mock. When the mock is called, it will always return the value assigned to this attribute. This is perfect for simulating successful function calls that return a predictable object.

```python
# test_return_value.py
from unittest.mock import Mock

def test_return_value_example():
    mock_func = Mock()
    mock_func.return_value = 42

    result = mock_func("some", "arguments")
    
    assert result == 42
    mock_func.assert_called_once_with("some", "arguments")
```

### `side_effect`: For Complex Behavior

`side_effect` is more versatile and powerful. It can be set to an exception, an iterable, or a callable function.

#### 1. Raising Exceptions

To test your code's error handling, you can set `side_effect` to an exception class or instance. When the mock is called, it will raise that exception.

```python
# test_side_effect.py
import pytest
from unittest.mock import Mock

def function_that_handles_errors(api_call):
    try:
        return api_call()
    except ValueError:
        return "Handled ValueError"

def test_side_effect_with_exception():
    mock_api = Mock()
    mock_api.side_effect = ValueError("Invalid API Key")

    result = function_that_handles_errors(mock_api)

    assert result == "Handled ValueError"
    mock_api.assert_called_once()
```

This is exactly what we did in our `test_format_weather_report_api_error` test to simulate a network failure.

#### 2. Returning a Sequence of Values

If you need a mock to return different values on subsequent calls, you can assign an iterable (like a list or tuple) to `side_effect`.

```python
# test_side_effect.py
from unittest.mock import Mock

def test_side_effect_with_iterable():
    # Imagine a mock for a database cursor's fetchone() method
    mock_fetchone = Mock()
    mock_fetchone.side_effect = [
        ("user1", "user1@a.com"),
        ("user2", "user2@b.com"),
        None  # The last call returns None, indicating no more rows
    ]

    assert mock_fetchone() == ("user1", "user1@a.com")
    assert mock_fetchone() == ("user2", "user2@b.com")
    assert mock_fetchone() is None
    
    # If you call it again after exhaustion, it raises StopIteration
    with pytest.raises(StopIteration):
        mock_fetchone()
```

#### 3. Using a Callable for Dynamic Behavior

For the most complex scenarios, you can assign a function (or any callable) to `side_effect`. The mock will delegate the call to your function, passing along any arguments it received. Your function's return value will be used as the mock's return value.

This is useful when the mock's output needs to depend on its input.

```python
# test_side_effect.py
from unittest.mock import Mock

def user_db_side_effect(user_id):
    """A fake function to simulate a user database."""
    if user_id == 1:
        return {"name": "Alice"}
    if user_id == 2:
        return {"name": "Bob"}
    return None

def test_side_effect_with_callable():
    mock_get_user = Mock()
    mock_get_user.side_effect = user_db_side_effect

    alice = mock_get_user(1)
    bob = mock_get_user(2)
    charlie = mock_get_user(3)

    assert alice == {"name": "Alice"}
    assert bob == {"name": "Bob"}
    assert charlie is None

    # You can still assert how the mock was called
    assert mock_get_user.call_count == 3
    mock_get_user.assert_any_call(1)
    mock_get_user.assert_any_call(2)
```

## Combining Mocks and Fixtures

## Combining Mocks and Fixtures

Using `@patch` is great, but it can become unwieldy if you need the same mock setup for multiple tests, or if you need to patch several things at once.

```python
# The "crowded decorator" problem
@patch("src.module.api.get_user")
@patch("src.module.api.get_account")
@patch("src.module.db.save_record")
def test_something(mock_save, mock_get_account, mock_get_user):
    # ... test logic ...
```

This is where fixtures, the foundation of pytest, come to the rescue. We can encapsulate our patching logic inside a fixture to create clean, reusable, and composable mock setups. This is the "pytest way" of handling mocks.

### The Wrong Way: Repetitive Setup

First, let's see the problem. Imagine we have several tests that all need a mock of our weather API that returns a "Sunny" response. We could copy-paste the setup, but that's not maintainable.

```python
# Repetitive setup (don't do this)
@patch("src.weather_reporter.requests.get")
def test_report_for_sunny_day(mock_requests_get):
    mock_response = mock_requests_get.return_value
    mock_response.json.return_value = {"temperature": 25, "condition": "Sunny"}
    # ... test logic ...

@patch("src.weather_reporter.requests.get")
def test_another_feature_on_a_sunny_day(mock_requests_get):
    mock_response = mock_requests_get.return_value
    mock_response.json.return_value = {"temperature": 25, "condition": "Sunny"}
    # ... another test's logic ...
```

### The Right Way: A Mocking Fixture

Let's refactor this into a fixture. We can use `patch` as a context manager inside the fixture.

```python
# tests/conftest.py
import pytest
from unittest.mock import patch

@pytest.fixture
def mock_sunny_weather_api():
    """
    A fixture that patches requests.get and mocks a sunny weather response.
    It yields the mock object for optional further configuration in tests.
    """
    # The 'with' statement starts the patch
    with patch("src.weather_reporter.requests.get") as mock_get:
        mock_response = mock_get.return_value
        mock_response.status_code = 200
        mock_response.json.return_value = {
            "temperature": 25,
            "condition": "Sunny"
        }
        yield mock_get  # The test runs here
    # The patch is automatically stopped when the 'with' block exits
```

```python
# tests/test_weather_reporter_with_fixture.py
from src.weather_reporter import format_weather_report

def test_format_weather_report_with_fixture(mock_sunny_weather_api):
    """
    Test the formatter using our fixture. The test is now much cleaner.
    """
    report = format_weather_report("London")
    assert report == "The weather in London is 25°C and Sunny."

    # We can still make assertions on the mock yielded by the fixture
    mock_sunny_weather_api.assert_called_once_with(
        "https://api.weather.com/data?city=London"
    )
```

This approach is vastly superior:

*   **DRY (Don't Repeat Yourself):** The mock setup is defined in one place. If the API response format changes, we only need to update the fixture.
*   **Readability:** The test function is now focused purely on the logic it's testing: call the function and assert the result. The setup mechanism is abstracted away.
*   **Reusability:** Any test in our suite can now request the `mock_sunny_weather_api` fixture to get a consistent testing environment.

### Using Pytest's `monkeypatch` Fixture

Pytest provides its own built-in fixture for modifying code during tests called `monkeypatch`. It's an alternative to `unittest.mock.patch` for some use cases.

`monkeypatch` is excellent for replacing attributes, dictionary items, or environment variables. Its most common method is `setattr`.

Let's rewrite our fixture using `monkeypatch`. This requires a bit more manual work, as `monkeypatch` doesn't create a `Mock` object for us, but it's very explicit.

```python
# tests/conftest.py
import pytest
from unittest.mock import Mock

@pytest.fixture
def mock_sunny_weather_api_monkeypatch(monkeypatch):
    """
    A fixture that uses monkeypatch to replace requests.get.
    """
    # We need to create the mock object ourselves
    mock_get = Mock()
    mock_response = mock_get.return_value
    mock_response.status_code = 200
    mock_response.json.return_value = {
        "temperature": 25,
        "condition": "Sunny"
    }
    
    # Use monkeypatch.setattr to replace the real function with our mock
    monkeypatch.setattr("src.weather_reporter.requests.get", mock_get)
    
    # Unlike the context manager, we don't need to yield.
    # Monkeypatch handles the teardown automatically.
    # We can return the mock if tests need to inspect it.
    return mock_get
```

### `patch` vs. `monkeypatch`: Which to Choose?

Both `unittest.mock.patch` and pytest's `monkeypatch` are excellent tools. Here's a general guideline:

*   Use **`unittest.mock.patch`** (as a decorator or context manager) when your primary goal is to replace an object with a `Mock` to verify interactions (e.g., `assert_called_with`). It's the standard for classic mocking.
*   Use **`monkeypatch`** when you want to replace something with a simple value or a fake function that doesn't need complex mock assertions. It's great for setting environment variables (`monkeypatch.setenv`), changing constants, or replacing a function with a simple lambda that returns a fixed value.

For most of the mocking scenarios you'll encounter, combining `unittest.mock.patch` with pytest fixtures provides the cleanest and most powerful pattern. It leverages the strengths of both libraries: the sophisticated mocking capabilities of `unittest.mock` and the elegant dependency injection and setup/teardown management of pytest fixtures.
