# Chapter 11: Testing Asynchronous Code

## The Basics of Async/Await

## The Basics of Async/Await

Before we can test asynchronous code, we need a solid mental model of what it is and why it exists. Modern applications spend a lot of time waiting: waiting for a database query to return, for a web API to respond, or for a file to be read from a disk. This is called **I/O-bound** work.

Imagine a chef in a kitchen. A synchronous chef would:
1.  Put a pot of water on the stove to boil.
2.  Stare at the pot until it boils.
3.  Chop vegetables.

This is incredibly inefficient. The chef is blocked, doing nothing, while waiting for the water.

An asynchronous chef works differently:
1.  Put a pot of water on the stove to boil.
2.  While the water is heating up, start chopping vegetables.
3.  When the water boils, pause chopping and pour the pasta in.
4.  While the pasta cooks, finish chopping the vegetables.

The asynchronous chef can manage multiple tasks by switching between them whenever one is "waiting." This is the core idea behind Python's `asyncio`. The keywords `async` and `await` are the tools that let us write code like the asynchronous chef.

-   `async def`: This defines a function as a **coroutine**. It's a special kind of function that can be paused and resumed. Putting water on the stove is like starting a coroutine.
-   `await`: This keyword says, "This is a waiting point. Pause this coroutine here, and let the event loop run other tasks until this one is ready to continue." This is the chef turning to chop vegetables while the water heats up.

Let's see this in a simple Python example. We'll create a function that simulates fetching data from a slow network API.

```python
# src/data_fetcher.py
import asyncio

async def fetch_data(source: str) -> dict:
    """Simulates fetching data from a slow source."""
    print(f"Start fetching from {source}...")
    # asyncio.sleep is the asynchronous version of time.sleep.
    # It pauses the coroutine without blocking the entire program.
    await asyncio.sleep(1)
    print(f"Done fetching from {source}.")
    return {"source": source, "data": [1, 2, 3]}

async def main():
    """A main function to run our coroutine."""
    result = await fetch_data("API Server")
    print(f"Received: {result}")

if __name__ == "__main__":
    # asyncio.run() starts the event loop and runs the coroutine until it's complete.
    asyncio.run(main())
```

If you run this file, you'll see the output with a 1-second delay, just as you'd expect.

```bash
$ python src/data_fetcher.py
Start fetching from API Server...
Done fetching from API Server.
Received: {'source': 'API Server', 'data': [1, 2, 3]}
```

The key takeaway is that an `async def` function doesn't run like a normal function. Calling it just creates a coroutine object. You need an event loop runner (like `asyncio.run()`) to actually execute it. This has a major implication for testing: vanilla pytest doesn't know how to run these coroutine objects.

## Testing Coroutines with pytest-asyncio

## Testing Coroutines with pytest-asyncio

Let's try to write a test for our `fetch_data` function using what we know so far.

### The Problem: Pytest Doesn't Speak Async Natively

If you write a test like a regular function call, you'll get a warning, and the test won't actually run the asynchronous code.

```python
# tests/test_data_fetcher_problem.py
from src.data_fetcher import fetch_data

def test_fetch_data_incorrectly():
    # This just creates a coroutine object, it doesn't run it!
    coro = fetch_data("API Server")
    # This test will pass, but it doesn't actually test the coroutine's logic.
    # You'll get a RuntimeWarning from pytest.
    assert coro is not None
```

Running this gives you a warning because the coroutine was never `await`ed.

```bash
============================= warnings summary =============================
tests/test_data_fetcher_problem.py::test_fetch_data_incorrectly
  /path/to/tests/test_data_fetcher_problem.py:6: RuntimeWarning: coroutine 'fetch_data' was never awaited
    coro = fetch_data("API Server")

-- Docs: https://docs.pytest.org/en/stable/how-to/capture-warnings.html
======================== 1 passed, 1 warning in 0.01s ========================
```

The test passed, but it didn't do what we wanted. It only tested that the function call created an object, not that the object, when run, produces the correct result.

### The Solution: `pytest-asyncio`

To bridge this gap, we use a powerful plugin called `pytest-asyncio`. It automatically handles the asyncio event loop for your tests.

First, install it:

```bash
pip install pytest-asyncio
```

Now, we can write our test correctly. We need to do two things:
1.  Define the test function with `async def`.
2.  Mark it with `@pytest.mark.asyncio`.

This marker tells `pytest-asyncio` that this is an asynchronous test that needs an event loop to run. Inside the test, we can now use the `await` keyword.

```python
# tests/test_data_fetcher.py
import pytest
from src.data_fetcher import fetch_data

@pytest.mark.asyncio
async def test_fetch_data_successfully():
    """Tests that our async function runs and returns the expected dict."""
    # Now we can use 'await' inside our test function
    result = await fetch_data("Test Source")

    assert isinstance(result, dict)
    assert result["source"] == "Test Source"
    assert result["data"] == [1, 2, 3]
```

Let's run this test.

```bash
$ pytest tests/test_data_fetcher.py
============================= test session starts ==============================
...
collected 1 item

tests/test_data_fetcher.py::test_fetch_data_successfully
Start fetching from Test Source...
Done fetching from Test Source.
PASSED                               [100%]

============================== 1 passed in 1.02s ===============================
```

Success! The `pytest-asyncio` plugin detected the marker, set up an event loop, ran our `await fetch_data(...)` call, and then tore down the loop. We have successfully tested our first coroutine. The "magic" is simply the plugin providing the `asyncio.run()` equivalent for each test function marked appropriately.

## Fixtures for Async Tests

## Fixtures for Async Tests

Just like regular tests, async tests need setup and teardown logic. Pytest fixtures work seamlessly with `pytest-asyncio`, and they can even be asynchronous themselves. This is incredibly useful for setting up resources that require async operations, like a database connection pool or an API client session.

### Asynchronous Fixtures

To create an asynchronous fixture, you simply define it with `async def`. You can then `await` other coroutines inside it. Tests that use this fixture must also be `async def` functions.

```python
# tests/conftest.py
import pytest
import asyncio

class AsyncDatabaseClient:
    """A mock async database client for demonstration."""
    def __init__(self, db_name):
        self._db_name = db_name
        self._is_connected = False

    async def connect(self):
        """Simulate an async connection."""
        await asyncio.sleep(0.1)
        self._is_connected = True
        print(f"\nConnected to {self._db_name}!")

    async def disconnect(self):
        """Simulate an async disconnection."""
        await asyncio.sleep(0.1)
        self._is_connected = False
        print(f"\nDisconnected from {self._db_name}!")

    async def query(self, sql):
        if not self._is_connected:
            raise ConnectionError("Database is not connected")
        await asyncio.sleep(0.2)
        return [{"id": 1, "name": "test_user"}]

@pytest.fixture
async def db_client():
    """An async fixture to set up and tear down a database client."""
    client = AsyncDatabaseClient("test_db")
    await client.connect()
    yield client
    await client.disconnect()
```

Here, our `db_client` fixture is a coroutine. It creates a client, `await`s its connection, `yield`s the client to the test, and then `await`s its disconnection after the test is complete.

Now, let's use this fixture in a test.

```python
# tests/test_async_db.py
import pytest

@pytest.mark.asyncio
async def test_database_query(db_client):
    """
    Tests a query using the async db_client fixture.
    The test function must be async to use an async fixture.
    """
    result = await db_client.query("SELECT * FROM users;")
    assert result == [{"id": 1, "name": "test_user"}]
```

When you run this test, you'll see the connect and disconnect messages from the fixture, proving the setup and teardown logic ran correctly around the test.

```bash
$ pytest -v -s tests/test_async_db.py
============================= test session starts ==============================
...
collected 1 item

tests/test_async_db.py::test_database_query
Connected to test_db!
PASSED
Disconnected from test_db!

============================== 1 passed in 0.43s ===============================
```

### Using Synchronous Fixtures in Async Tests

You are not required to make all your fixtures async. Regular, synchronous fixtures work perfectly fine when passed to an async test.

```python
# tests/conftest.py (add this fixture)

@pytest.fixture
def user_payload():
    """A regular, synchronous fixture."""
    return {"username": "test_user", "email": "test@example.com"}

# tests/test_async_db.py (add this test)

@pytest.mark.asyncio
async def test_with_sync_and_async_fixtures(db_client, user_payload):
    """
    This async test uses both an async fixture (db_client)
    and a sync fixture (user_payload).
    """
    print(f"Testing with payload: {user_payload}")
    result = await db_client.query("SELECT * FROM users;")
    assert result is not None
```

This works exactly as you'd expect. Pytest resolves all fixtures first—awaiting the async ones—before running the async test body. The rule is simple: if your fixture needs to `await` something, make it `async def`. Otherwise, a regular `def` is fine.

## Mocking Async Functions

## Mocking Async Functions

Mocking is essential for isolating the code you're testing from its dependencies, especially slow or unreliable ones like network calls. However, mocking asynchronous functions presents a new challenge: the mock object itself must be "awaitable."

### The Problem: A Regular Mock Isn't Awaitable

Let's go back to our `fetch_data` example. Imagine we have another function that uses it.

```python
# src/data_processor.py
from .data_fetcher import fetch_data

async def process_data_from_source(source: str) -> str:
    """Fetches and processes data."""
    data = await fetch_data(source)
    # In a real app, more complex processing would happen here.
    return f"Processed data for {data['source']} with {len(data['data'])} items."
```

To test `process_data_from_source` in isolation, we must mock `fetch_data`. Let's see what happens if we try to use a standard `unittest.mock.Mock`.

```python
# tests/test_data_processor_problem.py
import pytest
from unittest.mock import patch, Mock
from src.data_processor import process_data_from_source

@pytest.mark.asyncio
async def test_process_data_with_regular_mock():
    mock_fetch = Mock(
        return_value={"source": "mock", "data": [1, 2]}
    )
    with patch("src.data_processor.fetch_data", new=mock_fetch):
        # This will fail!
        result = await process_data_from_source("any_source")
        assert result == "Processed data for mock with 2 items."
```

Running this test results in a `TypeError`.

```
>       result = await process_data_from_source("any_source")
...
>       data = await fetch_data(source)
E       TypeError: object Mock can't be used in 'await' expression
```

The error message is crystal clear. The `await` keyword expects an awaitable object (like a coroutine), but we gave it a regular `Mock` instance.

### The Solution: `AsyncMock`

Since Python 3.8, the standard `unittest.mock` library includes `AsyncMock`, which is designed specifically for this purpose. An `AsyncMock` instance is awaitable, and it provides special assertion methods like `assert_awaited_once`.

```python
# tests/test_data_processor.py
import pytest
from unittest.mock import patch, AsyncMock
from src.data_processor import process_data_from_source

@pytest.mark.asyncio
async def test_process_data_with_async_mock():
    # Create an AsyncMock. It's awaitable!
    # We set the return value of the *awaited* mock.
    mock_fetch = AsyncMock(
        return_value={"source": "mock", "data": [1, 2]}
    )

    # Patch the function in the module where it is *used*.
    with patch("src.data_processor.fetch_data", new=mock_fetch):
        result = await process_data_from_source("any_source")
        assert result == "Processed data for mock with 2 items."

    # You can assert that the async mock was awaited.
    mock_fetch.assert_awaited_once()
    mock_fetch.assert_awaited_once_with("any_source")
```

This test now passes. By replacing `Mock` with `AsyncMock`, we provide an object that satisfies the `await` expression, allowing our test to run correctly while still isolating our function from the real `fetch_data` dependency.

### Autospeccing with `create_autospec`

A best practice when patching is to use `autospeccing` to ensure your mock has the same signature as the original object. This works perfectly with async functions too. `patch` will automatically use an `AsyncMock` if the target it's replacing is an `async` function.

```python
# tests/test_data_processor.py
import pytest
from unittest.mock import patch
from src.data_processor import process_data_from_source
from src.data_fetcher import fetch_data # Import for autospeccing

@pytest.mark.asyncio
async def test_process_data_with_autospec():
    # Using autospec=True is often the best approach.
    # It automatically creates an awaitable mock because fetch_data is an async def.
    with patch("src.data_processor.fetch_data", autospec=True) as mock_fetch:
        # The return value must be configured on the mock
        mock_fetch.return_value = {"source": "mock_spec", "data": [1, 2, 3]}

        result = await process_data_from_source("spec_source")
        assert result == "Processed data for mock_spec with 3 items."

    mock_fetch.assert_awaited_once_with("spec_source")
```

Using `autospec=True` is highly recommended. It prevents you from mocking a function with the wrong arguments, which can hide bugs. If the signature of `fetch_data` changes, this test will fail, alerting you to the necessary update.

## Common Pitfalls in Async Testing

## Common Pitfalls in Async Testing

Asynchronous programming introduces new kinds of errors. Being able to recognize them in your test failures will save you a lot of debugging time.

### Pitfall 1: Forgetting `await`

This is by far the most common mistake. You call a coroutine function but forget to `await` the result.

```python
# tests/test_pitfalls.py
import pytest
from src.data_fetcher import fetch_data

@pytest.mark.asyncio
async def test_forgot_await():
    # PITFALL: We called fetch_data but didn't await it.
    result_coro = fetch_data("some_source")

    # This assertion is on the coroutine object, not its result!
    # This test will fail, but for a confusing reason.
    assert await result_coro == {"source": "some_source", "data": [1, 2, 3]}
    # If the assertion was `assert result_coro is not None`, the test would
    # pass silently with a RuntimeWarning, which is even worse!
```

When you forget to `await` a call, you get a coroutine object back instead of the function's result. Your test might fail on a later assertion, or it might even pass if you're not careful, while printing a `RuntimeWarning: coroutine '...' was never awaited`.

**The Fix:** Always be vigilant about `await`ing any call to an `async def` function. If you see that warning, it's a red flag that your test isn't actually executing the code you think it is.

### Pitfall 2: Using Blocking I/O in Async Tests

The purpose of `asyncio` is to prevent the program from blocking on I/O. If you use a blocking call like `time.sleep()` instead of `asyncio.sleep()`, you defeat the purpose and can cause problems for the event loop.

```python
# tests/test_pitfalls.py
import pytest
import time
import asyncio

@pytest.mark.asyncio
async def test_blocking_sleep():
    start_time = time.monotonic()

    # PITFALL: Using time.sleep() blocks the entire event loop.
    # No other async tasks could run during this second.
    time.sleep(1)

    # This is the correct, non-blocking way.
    await asyncio.sleep(0.1)

    duration = time.monotonic() - start_time
    assert duration > 1.1
```

While this test will pass, using `time.sleep()` is an anti-pattern in async code. In a simple test, it's less harmful, but in a complex application, it can grind everything to a halt. Always use the `async` versions of libraries when working in an async context (e.g., `httpx` instead of `requests`, `asyncpg` instead of `psycopg2`).

### Pitfall 3: Manually Managing the Event Loop

`pytest-asyncio` provides an event loop for you. In almost all cases, you should not try to manage it yourself. Code like `loop = asyncio.get_event_loop()` or `asyncio.run()` inside a test is usually a sign that something is wrong.

```python
# tests/test_pitfalls.py
import pytest
import asyncio
from src.data_fetcher import fetch_data

@pytest.mark.asyncio
async def test_manual_loop_management_is_unnecessary():
    # PITFALL: You don't need to do this.
    # pytest-asyncio provides the loop.
    # In some complex cases this can even cause errors.
    loop = asyncio.get_running_loop()

    # The test body already runs inside the loop context.
    result = await fetch_data("source")
    assert result is not None
```

**The Fix:** Trust the plugin. Write your test code with `async def` and `await`, and let `pytest-asyncio` handle the loop setup and teardown. The only time you might need to interact with the loop is for advanced scheduling tasks, which is rare for most application testing.

## Testing Concurrent Code

## Testing Concurrent Code

The real power of `asyncio` shines when you run multiple I/O-bound tasks concurrently. A common pattern is to use `asyncio.gather()` to start several tasks and wait for them all to complete. Testing this is a great way to combine our knowledge of async tests and mocking.

Let's define a function that fetches data from multiple sources concurrently.

```python
# src/concurrent_fetcher.py
import asyncio
from .data_fetcher import fetch_data

async def fetch_all_data(sources: list[str]) -> list[dict]:
    """Fetches data from multiple sources concurrently."""
    tasks = []
    for source in sources:
        # Create a task for each coroutine. This schedules them on the event loop.
        task = asyncio.create_task(fetch_data(source))
        tasks.append(task)

    # asyncio.gather() waits for all tasks in the sequence to complete.
    results = await asyncio.gather(*tasks)
    return results
```

This function takes a list of sources, creates a "task" for each `fetch_data` call, and then uses `asyncio.gather` to run them all at the same time. If each `fetch_data` call takes 1 second, fetching three sources will still take only about 1 second, not 3.

How do we test this? We need to mock `fetch_data` in a way that allows us to provide different return values for different calls. The `side_effect` attribute of a mock is perfect for this.

```python
# tests/test_concurrent_fetcher.py
import pytest
from unittest.mock import patch, AsyncMock
from src.concurrent_fetcher import fetch_all_data

@pytest.mark.asyncio
async def test_fetch_all_data():
    sources = ["API_1", "API_2", "DB_1"]

    # We can use side_effect to return different values on each await.
    mock_results = [
        {"source": "API_1", "data": [1]},
        {"source": "API_2", "data": [2, 3]},
        {"source": "DB_1", "data": [4, 5, 6]},
    ]

    # Patch the function where it is used.
    with patch("src.concurrent_fetcher.fetch_data", new=AsyncMock()) as mock_fetch:
        mock_fetch.side_effect = mock_results

        # Run the function under test
        results = await fetch_all_data(sources)

    # Assert that the mock was awaited for each source
    assert mock_fetch.await_count == 3
    mock_fetch.assert_any_await("API_1")
    mock_fetch.assert_any_await("API_2")
    mock_fetch.assert_any_await("DB_1")

    # Assert that the results from gather are correct
    assert results == mock_results
```

This test effectively validates the logic of `fetch_all_data`:
1.  We patch the dependency, `fetch_data`, with an `AsyncMock`.
2.  We use `side_effect` to provide an iterable of return values. The mock will return the next item from the iterable each time it is awaited.
3.  We call our concurrent function.
4.  We assert that the mock was awaited three times, once for each source. `assert_any_await` is useful here because `gather` does not guarantee the execution order.
5.  Finally, we check that the aggregated result is what we expect.

This pattern gives you a powerful way to test the orchestration logic of your concurrent code without performing any actual I/O.
