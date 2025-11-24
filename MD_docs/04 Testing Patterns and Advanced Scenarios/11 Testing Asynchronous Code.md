# Chapter 11: Testing Asynchronous Code

## The Basics of Async/Await

## The Challenge of Asynchronous Code

Modern applications often perform tasks that don't require immediate computation, such as waiting for a network response, reading a file from a disk, or querying a database. In traditional synchronous programming, the entire application would block and wait for these I/O-bound operations to complete, wasting valuable CPU cycles.

Asynchronous programming, particularly with Python's `asyncio` library, solves this problem. It allows a program to start a long-running task (like a network request) and then switch to working on other tasks instead of waiting idly. When the long-running task is finished, the program can switch back to handle the result. This is managed by an **event loop**, which orchestrates the execution of these tasks.

### Core Concepts: `async` and `await`

Python provides two keywords to support this model: `async` and `await`.

1.  **`async def`**: This syntax is used to define a **coroutine function**. When you call a coroutine function, it doesn't execute immediately. Instead, it returns a **coroutine object**—a blueprint for the work that needs to be done.

2.  **`await`**: This keyword is used inside a coroutine function to pause its execution and hand control back to the event loop. You can only `await` other **awaitables** (like other coroutine objects or tasks). The event loop will then run another task. When the awaited operation is complete, the event loop will resume the paused coroutine right where it left off.

Let's see a simple example.

```python
# a_simple_async_program.py
import asyncio
import time

async def fetch_data(source: str, delay: int) -> str:
    """A coroutine that simulates a slow network call."""
    print(f"Fetching data from {source}...")
    await asyncio.sleep(delay)  # Pause here, let other tasks run
    print(f"Data fetched from {source}.")
    return f"Data from {source}"

async def main():
    """The main entry point for our async program."""
    start_time = time.time()
    print("Starting concurrent fetches.")

    # Schedule both coroutines to run concurrently on the event loop
    task1 = asyncio.create_task(fetch_data("API 1", 2))
    task2 = asyncio.create_task(fetch_data("Database", 3))

    # Now, wait for both tasks to complete
    result1 = await task1
    result2 = await task2

    print(f"Result 1: {result1}")
    print(f"Result 2: {result2}")

    end_time = time.time()
    print(f"Total time: {end_time - start_time:.2f} seconds.")

if __name__ == "__main__":
    # To run the top-level 'main' coroutine, we need to use asyncio.run()
    # This creates an event loop, runs the coroutine until it's done,
    # and then closes the loop.
    asyncio.run(main())
```

When you run this script, you'll see output like this:
```bash
$ python a_simple_async_program.py
Starting concurrent fetches.
Fetching data from API 1...
Fetching data from Database...
Data fetched from API 1.
Data fetched from Database.
Result 1: Data from API 1
Result 2: Data from Database
Total time: 3.01 seconds.
```
Notice that the total time is approximately 3 seconds (the duration of the longest task), not 5 seconds (2 + 3). This is the power of concurrency. Both `asyncio.sleep()` calls were running "at the same time."

The challenge for us is that pytest, by default, doesn't know how to handle `async def` functions. It doesn't know how to create an event loop or `await` the coroutine objects our tests will return. This is where we need a specialized plugin.

## Testing Coroutines with pytest-asyncio

## Phase 1: Establish the Reference Implementation

To explore async testing, we need a realistic piece of asynchronous code to test. We'll build a simple client for a hypothetical weather API. This is a perfect use case for `asyncio`, as its primary job is to make network requests, which are I/O-bound.

Our anchor example will be this `WeatherAPIClient`.

```python
# weather_client.py
import asyncio

class ExternalServiceError(Exception):
    """Custom exception for API failures."""
    pass

class WeatherAPIClient:
    """A simple client for a weather API."""

    async def get_current_temperature(self, city: str) -> float:
        """
        Fetches the current temperature for a given city.
        This simulates a network call that can fail.
        """
        print(f"Fetching temperature for {city}...")
        # Simulate a network delay
        await asyncio.sleep(1.0)

        if city == "error":
            raise ExternalServiceError("API request failed")

        # In a real app, this would be the result of a network call
        temp = 20.0 + len(city)
        print(f"Temperature for {city} is {temp}°C")
        return temp
```

### Iteration 0: The Naive Attempt

Let's try to write a pytest test for this client just like we would for any synchronous code. We'll define our test function using `async def` because it needs to `await` the method we're testing.

```python
# test_weather_client_v1.py
from weather_client import WeatherAPIClient

# This is a coroutine function
async def test_get_current_temperature_naive():
    """
    A naive attempt to test an async function.
    This will NOT work as expected.
    """
    client = WeatherAPIClient()
    temperature = await client.get_current_temperature("London")
    assert temperature == 20.0 + len("London")
```

Now, let's run this with pytest. The result is not a pass or a fail, but a warning.

```bash
$ pytest -q test_weather_client_v1.py
.
1 passed, 1 warning in 0.01s

=============================== warnings summary ===============================
test_weather_client_v1.py:4
  /path/to/test_weather_client_v1.py:4: PytestRemovedIn8Warning:
  Defining 'async def' tests without 'asyncio' marker is deprecated.
  Please add '@pytest.mark.asyncio' to all your async tests.
  See https://... for details.
    async def test_get_current_temperature_naive():

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
```

### Diagnostic Analysis: Reading the Failure

This output is subtle and dangerous. Pytest says `1 passed`, but it also gives a critical warning. In older versions of pytest, this test would silently do nothing. Modern versions provide this warning to guide you.

**The complete output**:
```bash
============================= test session starts ==============================
...
collected 1 item

test_weather_client_v1.py .                                              [100%]

=============================== warnings summary ===============================
test_weather_client_v1.py:4
  /path/to/test_weather_client_v1.py:4: PytestRemovedIn8Warning: Defining 'async def' tests without 'asyncio' marker is deprecated. Please add '@pytest.mark.asyncio' to all your async tests.
...
========================= 1 passed, 1 warning in ...s ==========================
```

**Let's parse this**:

1.  **The summary line**: `1 passed, 1 warning`. This is misleading. The test "passed" only because pytest didn't execute the coroutine, so no `AssertionError` could possibly be raised. The test did nothing.
2.  **The warning**: `PytestRemovedIn8Warning: Defining 'async def' tests without 'asyncio' marker is deprecated. Please add '@pytest.mark.asyncio' to all your async tests.` This is the key. Pytest recognizes the `async def` syntax but is telling us it doesn't have a built-in way to handle it. It's pointing us directly to the solution.

**Root cause identified**: Pytest's default test runner does not know how to run a coroutine. It sees the `async def` test, creates the coroutine object, but never `await`s it or runs it in an event loop.

**Why the current approach can't solve this**: We are missing the machinery to manage the `asyncio` event loop for our test function.

**What we need**: A pytest plugin that integrates `asyncio`, providing an event loop and properly running `async def` tests.

### Iteration 1: Introducing `pytest-asyncio`

The solution is the `pytest-asyncio` plugin. It's the standard tool for this job.

First, install it:

```bash
pip install pytest-asyncio
```

This plugin provides a marker, `@pytest.mark.asyncio`, which signals to pytest that the decorated test function is a coroutine and should be run inside an `asyncio` event loop.

Let's apply this fix to our test.

**Before:**

```python
# test_weather_client_v1.py (problematic)
from weather_client import WeatherAPIClient

async def test_get_current_temperature_naive():
    client = WeatherAPIClient()
    temperature = await client.get_current_temperature("London")
    assert temperature == 20.0 + len("London")
```

**After:**

```python
# test_weather_client_v2.py (solution)
import pytest
from weather_client import WeatherAPIClient, ExternalServiceError

@pytest.mark.asyncio
async def test_get_current_temperature():
    """Tests the successful retrieval of temperature."""
    client = WeatherAPIClient()
    temperature = await client.get_current_temperature("London")
    assert temperature == 25.0

@pytest.mark.asyncio
async def test_get_current_temperature_error():
    """Tests the case where the external API fails."""
    client = WeatherAPIClient()
    with pytest.raises(ExternalServiceError, match="API request failed"):
        await client.get_current_temperature("error")
```

Now, let's run the corrected tests.

```bash
$ pytest -v test_weather_client_v2.py
============================= test session starts ==============================
...
collected 2 items

test_weather_client_v2.py::test_get_current_temperature
Fetching temperature for London...
Temperature for London is 25.0°C
PASSED [ 50%]

test_weather_client_v2.py::test_get_current_temperature_error
Fetching temperature for error...
PASSED [100%]

============================== 2 passed in 2.05s ===============================
```

Success! The tests now run correctly. Notice the total time is around 2 seconds, because each test's `asyncio.sleep(1.0)` was executed. The `@pytest.mark.asyncio` marker instructed `pytest-asyncio` to:
1.  Create and manage an `asyncio` event loop for the test.
2.  Run the test coroutine (`test_get_current_temperature`) on that event loop until it completes.
3.  Handle the results (pass/fail) just like a regular test.

This solves our initial problem, but our tests are still creating a new `WeatherAPIClient` instance inside each function. This is a perfect use case for a fixture.

## Fixtures for Async Tests

## Iteration 2: Creating Asynchronous Fixtures

Our tests now work, but they repeat the `client = WeatherAPIClient()` setup. Let's refactor this into a fixture.

A synchronous fixture is straightforward:

```python
# A standard, synchronous fixture
@pytest.fixture
def client():
    return WeatherAPIClient()
```

But what if our fixture needs to perform asynchronous operations for setup or teardown? For example, it might need to establish an async database connection or authenticate with a service.

### The Problem: Fixtures that Need to be `async`

Let's imagine our `WeatherAPIClient` required some asynchronous initialization. We would need an `async def` fixture.

```python
# test_weather_client_v3.py
import pytest
from weather_client import WeatherAPIClient, ExternalServiceError

@pytest.fixture
async def client():
    """
    An async fixture to provide a WeatherAPIClient instance.
    This simulates async setup, e.g., acquiring a resource from a pool.
    """
    # In a real scenario, you might await an async connection function here.
    print("\n(Setting up async client)")
    await asyncio.sleep(0.01)
    yield WeatherAPIClient()
    # And here, you might await an async teardown function.
    print("\n(Tearing down async client)")
    await asyncio.sleep(0.01)

@pytest.mark.asyncio
async def test_get_current_temperature(client: WeatherAPIClient):
    temperature = await client.get_current_temperature("Tokyo")
    assert temperature == 25.0

@pytest.mark.asyncio
async def test_get_current_temperature_error(client: WeatherAPIClient):
    with pytest.raises(ExternalServiceError):
        await client.get_current_temperature("error")
```

Let's run this. You might expect this to fail, as a regular pytest runner wouldn't know how to handle an `async` fixture.

```bash
$ pytest -v -s test_weather_client_v3.py
============================= test session starts ==============================
...
collected 2 items

test_weather_client_v3.py::test_get_current_temperature
(Setting up async client)
Fetching temperature for Tokyo...
Temperature for Tokyo is 25.0°C
PASSED
(Tearing down async client)

test_weather_client_v3.py::test_get_current_temperature_error
(Setting up async client)
Fetching temperature for error...
PASSED
(Tearing down async client)

============================== 2 passed in 2.08s ===============================
```

It just works! This is one of the most powerful features of `pytest-asyncio`.

**How it works**: When a test is marked with `@pytest.mark.asyncio`, the plugin extends its magic to fixtures as well. If it encounters a fixture defined with `async def`, it will correctly run it within the same event loop as the test, `await`ing the setup part before the test runs and the teardown part after it completes.

This seamless integration allows you to build complex, asynchronous setup and teardown logic into your fixtures, keeping your tests clean and readable, without needing any special syntax beyond `async def`.

Our tests are now well-structured, but they are integration tests. They rely on the (simulated) behavior of an external service. To create true unit tests, we need to mock the I/O-bound part of our code.

## Mocking Async Functions

## Iteration 3: Unit Testing with Async Mocks

Our current tests for `get_current_temperature` are slow because of `asyncio.sleep(1.0)`. More importantly, they test the integration with a (simulated) external service. A unit test should isolate the code being tested from its dependencies. In our case, the dependency is the "network call."

Let's refactor our client slightly to make the network-facing part easier to mock.

```python
# weather_client_v2.py
import asyncio

class ExternalServiceError(Exception):
    """Custom exception for API failures."""
    pass

class WeatherAPIClient:
    """
    A refactored client with a separate method for data fetching,
    making it easier to mock.
    """
    async def _fetch_data(self, city: str) -> float:
        """Internal method simulating the actual network call."""
        await asyncio.sleep(1.0)
        if city == "error":
            raise ExternalServiceError("API request failed")
        return 20.0 + len(city)

    async def get_current_temperature(self, city: str) -> float:
        """Public method containing business logic."""
        print(f"Fetching temperature for {city}...")
        temp = await self._fetch_data(city)
        print(f"Temperature for {city} is {temp}°C")
        # Imagine more logic here, e.g., unit conversion, caching, etc.
        return temp
```

Our goal is to test `get_current_temperature` without actually executing `_fetch_data`.

### The Problem: Mocking a Coroutine

Let's try to mock `_fetch_data` using `unittest.mock.MagicMock` and `monkeypatch`, as we learned in Chapter 9.

```python
# test_weather_client_v4_failure.py
import pytest
from unittest.mock import MagicMock
from weather_client_v2 import WeatherAPIClient

@pytest.mark.asyncio
async def test_with_standard_mock(monkeypatch):
    """This test will fail!"""
    mock_response = 30.0
    # Create a standard mock
    mock_fetch = MagicMock(return_value=mock_response)

    # Patch the internal method
    monkeypatch.setattr(
        WeatherAPIClient,
        "_fetch_data",
        mock_fetch
    )

    client = WeatherAPIClient()
    # This 'await' will cause a TypeError
    temperature = await client.get_current_temperature("TestCity")
    assert temperature == mock_response
```

Running this test results in a `TypeError`.

```bash
$ pytest test_weather_client_v4_failure.py
============================= test session starts ==============================
...
collected 1 item

test_weather_client_v4_failure.py F                                      [100%]

=================================== FAILURES ===================================
_________________________ test_with_standard_mock __________________________

monkeypatch = <_pytest.monkeypatch.MonkeyPatch object at 0x...>

    @pytest.mark.asyncio
    async def test_with_standard_mock(monkeypatch):
        """This test will fail!"""
        mock_response = 30.0
        # Create a standard mock
        mock_fetch = MagicMock(return_value=mock_response)

        # Patch the internal method
        monkeypatch.setattr(
            WeatherAPIClient,
            "_fetch_data",
            mock_fetch
        )

        client = WeatherAPIClient()
        # This 'await' will cause a TypeError
>       temperature = await client.get_current_temperature("TestCity")

test_weather_client_v4_failure.py:22:
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
weather_client_v2.py:19: in get_current_temperature
    temp = await self._fetch_data(city)
E   TypeError: object MagicMock can't be used in 'await' expression
=========================== short test summary info ============================
FAILED test_weather_client_v4_failure.py::test_with_standard_mock - TypeError...
============================== 1 failed in 0.12s ===============================
```

### Diagnostic Analysis: Reading the Failure

**The complete output**: The traceback clearly points to the `await` line and ends with `TypeError: object MagicMock can't be used in 'await' expression`.

**Let's parse this section by section**:

1.  **The summary line**: `FAILED ... - TypeError: object MagicMock can't be used in 'await' expression`. This is incredibly descriptive. It tells us the exact type of error and what caused it.
2.  **The traceback**: It shows the call chain: our test called `get_current_temperature`, which in turn tried to `await self._fetch_data(city)`.
3.  **The assertion introspection**: Not applicable here as the error happened before the assertion.
4.  **The key line**: `E   TypeError: object MagicMock can't be used in 'await' expression`.

**Root cause identified**: The `await` keyword requires an "awaitable" object. A coroutine is an awaitable. A standard `MagicMock` instance is not. When our code tried to `await` the mock object returned by the patched `_fetch_data`, Python raised a `TypeError`.

**Why the current approach can't solve this**: `MagicMock` is designed for synchronous code. It doesn't have the necessary internal machinery (`__await__` method) to be compatible with the `await` keyword.

**What we need**: A special kind of mock object that is itself awaitable.

### The Solution: `unittest.mock.AsyncMock`

Since Python 3.8, the standard `unittest.mock` library includes `AsyncMock`, which is designed specifically for this purpose. An `AsyncMock` behaves like a `MagicMock` but is also an awaitable.

Let's fix our test using `AsyncMock`.

**Before:**

```python
# test_weather_client_v4_failure.py (problematic part)
from unittest.mock import MagicMock

# ...
mock_fetch = MagicMock(return_value=mock_response)
# ...
```

**After:**

```python
# test_weather_client_v4_solution.py
import pytest
from unittest.mock import AsyncMock
from weather_client_v2 import WeatherAPIClient, ExternalServiceError

@pytest.mark.asyncio
async def test_get_current_temperature_mocked(monkeypatch):
    """Tests the public method with the internal fetcher mocked."""
    # AsyncMock returns an awaitable that resolves to this value.
    mock_response = 30.0
    
    # Use AsyncMock instead of MagicMock
    mock_fetch = AsyncMock(return_value=mock_response)
    
    monkeypatch.setattr(WeatherAPIClient, "_fetch_data", mock_fetch)

    client = WeatherAPIClient()
    temperature = await client.get_current_temperature("TestCity")

    # Assert our logic works
    assert temperature == mock_response
    
    # Assert the mock was called correctly
    mock_fetch.assert_awaited_once_with("TestCity")

@pytest.mark.asyncio
async def test_get_current_temperature_mocked_error(monkeypatch):
    """Tests that exceptions from the fetcher are propagated."""
    # We can also mock side effects, like raising exceptions.
    mock_fetch = AsyncMock(side_effect=ExternalServiceError("Network down"))
    monkeypatch.setattr(WeatherAPIClient, "_fetch_data", mock_fetch)

    client = WeatherAPIClient()
    with pytest.raises(ExternalServiceError, match="Network down"):
        await client.get_current_temperature("TestCity")

    mock_fetch.assert_awaited_once_with("TestCity")
```

Let's run the corrected tests.

```bash
$ pytest -v test_weather_client_v4_solution.py
============================= test session starts ==============================
...
collected 2 items

test_weather_client_v4_solution.py::test_get_current_temperature_mocked
Fetching temperature for TestCity...
Temperature for TestCity is 30.0°C
PASSED [ 50%]

test_weather_client_v4_solution.py::test_get_current_temperature_mocked_error
Fetching temperature for TestCity...
PASSED [100%]

============================== 2 passed in 0.05s ===============================
```

The tests now pass, and they are lightning-fast! We have successfully isolated our `get_current_temperature` method from its I/O dependency.

Notice the new assertion methods provided by `AsyncMock`:
-   `mock.assert_awaited_once_with(...)`: Checks that the mock was awaited exactly once with the specified arguments.
-   `mock.await_count`: An attribute to check how many times the mock was awaited.
-   `mock.await_args`: The arguments for the last await.

This gives us full power to test the interactions with our asynchronous dependencies.

## Common Pitfalls in Async Testing

Asynchronous code introduces new patterns and, consequently, new ways for things to go wrong. Here are some of the most common failure modes you'll encounter when testing async code with pytest.

### Common Failure Modes and Their Signatures

#### Symptom: `RuntimeWarning: coroutine '...' was never awaited`

This is the classic mistake when starting with `asyncio`.

**Pytest output pattern**:
```bash
=============================== warnings summary ===============================
test_my_async.py::test_forgot_marker
  /path/to/test_my_async.py:5: PytestRemovedIn8Warning: Defining 'async def' tests without 'asyncio' marker is deprecated.
    async def test_forgot_marker():
```
In older versions or different contexts, you might see:
```
RuntimeWarning: coroutine 'test_forgot_marker' was never awaited
```

**Diagnostic clues**:
-   The test "passes" but seems to do nothing.
-   A `PytestRemovedIn8Warning` or `RuntimeWarning` is present in the output.
-   Breakpoints inside the test are never hit.

**Root cause**: You have defined an `async def` test but have forgotten to add the `@pytest.mark.asyncio` marker. Pytest calls the function, gets a coroutine object back, and then discards it without running it.

**Solution**: Add `@pytest.mark.asyncio` to the test function.

#### Symptom: `TypeError: object MagicMock can't be used in 'await' expression`

We just saw this one in detail. It's what happens when you mix synchronous mocks with asynchronous code.

**Pytest output pattern**:
```bash
_________________________ test_with_standard_mock __________________________
...
E   TypeError: object MagicMock can't be used in 'await' expression
```

**Diagnostic clues**:
-   The traceback points to a line with an `await` keyword.
-   The object being awaited is a `MagicMock` or `Mock` instance.

**Root cause**: You are trying to `await` a standard `unittest.mock.MagicMock` object, which is not an awaitable.

**Solution**: Replace `MagicMock` with `unittest.mock.AsyncMock`.

#### Symptom: Calling a regular function with `await`

This is the reverse of the previous problem.

**Pytest output pattern**:
```bash
____________________________ test_awaiting_sync ____________________________
...
    async def my_test():
>       await my_sync_function()
E       TypeError: object NoneType can't be used in 'await' expression
```
*(Note: The type might be different from `NoneType` depending on what the sync function returns.)*

**Diagnostic clues**:
-   A `TypeError` occurs on an `await` line.
-   The function being awaited was defined with `def`, not `async def`.

**Root cause**: The `await` keyword can only be used with awaitable objects (like coroutines). You are trying to use it on the return value of a regular synchronous function.

**Solution**: Ensure you only `await` functions defined with `async def`. If you need to call a synchronous function that is blocking, you may need to run it in a separate thread pool using `asyncio.to_thread` (Python 3.9+) to avoid blocking the event loop.

## Testing Concurrent Code

## Iteration 4: Testing `asyncio.gather`

A major benefit of `asyncio` is running multiple I/O-bound tasks concurrently. A common pattern is to use `asyncio.gather` to schedule and await multiple coroutines at once.

Let's add a new method to our `WeatherAPIClient` that fetches temperatures for several cities concurrently.

```python
# weather_client_v3.py
import asyncio

class ExternalServiceError(Exception):
    pass

class WeatherAPIClient:
    async def _fetch_data(self, city: str) -> float:
        # This remains our "external dependency"
        await asyncio.sleep(0.1) # Reduced sleep for faster example
        if city == "error":
            raise ExternalServiceError("API request failed")
        return 20.0 + len(city)

    async def get_current_temperature(self, city: str) -> float:
        return await self._fetch_data(city)

    async def get_temperatures_for_cities(self, cities: list[str]) -> dict[str, float]:
        """
        Fetches temperatures for multiple cities concurrently.
        """
        tasks = [self.get_current_temperature(city) for city in cities]
        results = await asyncio.gather(*tasks)
        return dict(zip(cities, results))
```

How do we test `get_temperatures_for_cities`? We want to verify two things:
1.  It returns the correct dictionary of cities and temperatures.
2.  It calls `get_current_temperature` for each city.

Mocking is essential here. We don't want to wait for multiple `asyncio.sleep` calls. We also need our mock to return different values for different inputs.

### The Technique: `AsyncMock` with `side_effect`

The `side_effect` attribute of a mock is perfect for this. We can assign it a function (or a dictionary) that determines the return value based on the arguments the mock was called with.

Let's write a test for our new concurrent method.

```python
# test_weather_client_v5.py
import pytest
from unittest.mock import AsyncMock
from weather_client_v3 import WeatherAPIClient

@pytest.mark.asyncio
async def test_get_temperatures_for_cities_concurrently(monkeypatch):
    """
    Tests that the concurrent fetching method works correctly.
    """
    # Define a mapping of inputs to desired outputs for our mock
    mock_responses = {
        "London": 25.0,
        "Tokyo": 28.0,
        "Cairo": 35.0,
    }

    # The side_effect function will be called with the same args as the mock
    async def mock_get_temp(city: str):
        return mock_responses.get(city, 0.0)

    # We patch the method that is called concurrently
    monkeypatch.setattr(
        WeatherAPIClient,
        "get_current_temperature",
        AsyncMock(side_effect=mock_get_temp)
    )

    client = WeatherAPIClient()
    cities_to_test = ["London", "Tokyo", "Cairo"]
    
    # Run the method under test
    temperatures = await client.get_temperatures_for_cities(cities_to_test)

    # Verify the final result
    expected_temperatures = {
        "London": 25.0,
        "Tokyo": 28.0,
        "Cairo": 35.0,
    }
    assert temperatures == expected_temperatures

    # Verify the interactions with the mock
    mocked_method = WeatherAPIClient.get_current_temperature
    assert mocked_method.await_count == 3
    
    # Check that it was called with all the cities we expected
    # Note: asyncio.gather does not guarantee call order, so we check unsorted.
    called_cities = [call.args[0] for call in mocked_method.await_args_list]
    assert sorted(called_cities) == sorted(cities_to_test)
```

Let's run this test.

```bash
$ pytest -v test_weather_client_v5.py
============================= test session starts ==============================
...
collected 1 item

test_weather_client_v5.py::test_get_temperatures_for_cities_concurrently PASSED [100%]

============================== 1 passed in 0.03s ===============================
```

The test passes instantly. By mocking the coroutine called inside `asyncio.gather`, we can test the orchestrating logic of `get_temperatures_for_cities` without performing any real I/O or waiting. The `side_effect` gives us precise control over the behavior of our dependency, allowing us to simulate complex scenarios with ease.

### The Journey: From Problem to Solution

| Iteration | Failure Mode / Problem                                 | Technique Applied                     | Result                                                              |
| --------- | ------------------------------------------------------ | ------------------------------------- | ------------------------------------------------------------------- |
| 0         | `RuntimeWarning: coroutine was never awaited`          | None (Naive `async def` test)         | Test does not run, gives a warning.                                 |
| 1         | Pytest doesn't have an asyncio event loop.             | `pip install pytest-asyncio`, `@pytest.mark.asyncio` | Async tests run correctly.                                          |
| 2         | How to handle async setup/teardown?                    | `async def` fixtures                  | Fixtures can perform async operations seamlessly.                   |
| 3         | `TypeError` when mocking coroutines with `MagicMock`.  | `unittest.mock.AsyncMock`             | Coroutines can be mocked, enabling fast unit tests.                 |
| 4         | How to test concurrent `asyncio.gather` calls?         | `AsyncMock(side_effect=...)`          | Logic that orchestrates multiple coroutines can be tested reliably. |

### Final Implementation

Here is the final, robust set of tests for our `WeatherAPIClient`, incorporating all the techniques we've learned.

```python
# test_weather_client_final.py
import pytest
import asyncio
from unittest.mock import AsyncMock
from weather_client_v3 import WeatherAPIClient, ExternalServiceError

# An async fixture for our client
@pytest.fixture
async def client():
    # Simulate async setup if needed
    await asyncio.sleep(0.01)
    return WeatherAPIClient()

# Test for the single-city method (mocked)
@pytest.mark.asyncio
async def test_get_current_temperature_mocked(monkeypatch):
    mock_fetch = AsyncMock(return_value=30.0)
    monkeypatch.setattr(WeatherAPIClient, "_fetch_data", mock_fetch)

    client = WeatherAPIClient()
    temperature = await client.get_current_temperature("TestCity")

    assert temperature == 30.0
    mock_fetch.assert_awaited_once_with("TestCity")

# Test for the concurrent method (mocked)
@pytest.mark.asyncio
async def test_get_temperatures_for_cities_mocked(monkeypatch):
    mock_responses = {"London": 25.0, "Tokyo": 28.0}
    async def mock_get_temp(city: str):
        return mock_responses.get(city)

    monkeypatch.setattr(
        WeatherAPIClient,
        "get_current_temperature",
        AsyncMock(side_effect=mock_get_temp)
    )

    client = WeatherAPIClient()
    cities = ["London", "Tokyo"]
    temperatures = await client.get_temperatures_for_cities(cities)

    assert temperatures == {"London": 25.0, "Tokyo": 28.0}
    
    mocked_method = WeatherAPIClient.get_current_temperature
    assert mocked_method.await_count == 2

# An integration test (unmocked) using the fixture
@pytest.mark.asyncio
async def test_get_current_temperature_integration(client: WeatherAPIClient):
    # This test performs the actual asyncio.sleep
    temperature = await client.get_current_temperature("London")
    assert temperature == 26.0 # 20.0 + len("London")
```

### Lessons Learned

Testing asynchronous code in Python might seem daunting, but the principles are a direct extension of synchronous testing, with a few key additions:

1.  **Embrace the Plugin**: Don't fight the framework. `pytest-asyncio` is the standard and solves the core problem of integrating `asyncio`'s event loop with pytest's runner. Use `@pytest.mark.asyncio` for all your async tests.
2.  **Async Fixtures are Free**: `pytest-asyncio` makes `async def` fixtures work just like regular ones, allowing you to build clean, asynchronous test setups.
3.  **Use the Right Mock**: For any coroutine you need to mock, `unittest.mock.AsyncMock` is the tool. It prevents `TypeError`s and provides async-specific assertions like `assert_awaited_once_with`.
4.  **Isolate Concurrency**: When testing functions that use `asyncio.gather`, mock the underlying concurrent tasks. This lets you test your orchestration logic quickly and reliably without performing the actual concurrent I/O.
