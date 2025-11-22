# Chapter 9: Spies, Stubs, and Test Doubles

## The Difference Between Mocks, Stubs, and Spies

## What Are Test Doubles?

In the previous chapter, we introduced `unittest.mock` and the concept of mocking. "Mock" is often used as a catch-all term, but it's technically one of several types of objects that stand in for real objects during testing. The general, more accurate term for these stand-ins is **Test Double**.

Think of a stunt double in a movie. They look and act like the real actor for a specific scene, allowing the production to film something that would be dangerous or impractical for the star. Test doubles do the same for your code: they stand in for components that are slow, unreliable, or have side effects (like databases, external APIs, or the file system), allowing your tests to run quickly and in isolation.

There are several kinds of test doubles, but we'll focus on the three most common and useful ones: Stubs, Spies, and Mocks. Understanding their distinct roles will make your tests clearer and more intentional.

### Stubs: The State Verifiers

A **Stub** is a simple object that provides canned answers to calls made during the test. It's all about **state**. You use a stub when your test needs a dependency to return specific data, but you don't care *how* or *if* the dependency was called.

Imagine a function that fetches user data and then formats it. To test the formatting logic, you don't need a real database. You just need an object that returns a predictable user dictionary.

Let's look at a system that needs a stub.

```python
# src/user_service.py

def get_user_data(user_id):
    """In a real app, this would query a database."""
    # For this example, we'll simulate a slow or complex lookup.
    raise ConnectionError("Cannot connect to the database!")

def format_user_display(user_id):
    """Fetches user data and formats it for display."""
    user = get_user_data(user_id)
    return f"{user['name']} ({user['email']})"
```

Testing `format_user_display` directly is impossible because `get_user_data` will raise an error. We need to replace `get_user_data` with a stub that returns a predictable dictionary.

```python
# tests/test_user_service_stub.py
from unittest.mock import patch
from src.user_service import format_user_display

def test_format_user_display_with_stub():
    # This is our Stub. It's pre-programmed to return a specific dictionary.
    # We don't care how many times it's called, only that it provides state.
    stub_user_data = {"name": "Alice", "email": "alice@example.com"}

    # We use patch to replace the real get_user_data with our stub's behavior.
    with patch('src.user_service.get_user_data', return_value=stub_user_data) as mock_get_user:
        # The test now runs against the stub, not the real function.
        display_name = format_user_display(user_id=1)

    # We assert the result of our formatting logic.
    assert display_name == "Alice (alice@example.com)"

    # Note: We don't assert anything about the mock_get_user itself.
    # We only used it to provide state.
```

The key takeaway: A stub's job is to provide data to the system under test. The test verifies the system's output, not the stub's interactions.

### Spies: The Behavior Verifiers

A **Spy** is a test double that records what happens to it. It's all about **behavior**. You use a spy when you need to verify that a certain function was called, how many times it was called, or what arguments it was called with. The return value is often unimportant.

Imagine a function that processes a payment and then sends a notification email. You want to test that the email function is actually called with the correct details.

Let's define a system that needs a spy.

```python
# src/notifications.py

def send_notification(email, message):
    """In a real app, this would connect to an SMTP server and send an email."""
    print(f"Sending email to {email}: {message}")
    # This has a side effect we want to avoid in tests.
    return "Success"

def process_order(order_id, customer_email):
    """Processes an order and notifies the customer."""
    # Some complex order processing logic would go here...
    print(f"Processing order {order_id}...")

    # After processing, send a notification.
    message = f"Your order {order_id} has been processed."
    send_notification(customer_email, message)
    return True
```

We don't want to actually send an email during our test. We just want to *spy* on the `send_notification` function to ensure it was called correctly.

```python
# tests/test_notifications_spy.py
from unittest.mock import patch
from src.notifications import process_order

def test_process_order_sends_notification():
    # We patch send_notification. The resulting mock object acts as our Spy.
    # It will record all calls made to it.
    with patch('src.notifications.send_notification') as spy_send_notification:
        process_order(order_id="ABC-123", customer_email="bob@example.com")

    # Now we use the spy to verify behavior.
    # Was the function called exactly once?
    spy_send_notification.assert_called_once()

    # Was it called with the correct arguments?
    expected_message = "Your order ABC-123 has been processed."
    spy_send_notification.assert_called_once_with(
        "bob@example.com",
        expected_message
    )
```

The key takeaway: A spy's job is to record interactions. The test uses the spy's records to verify that the system under test behaved as expected.

### Mocks: The All-in-One

A **Mock** is the most powerful type of test double. It combines the capabilities of stubs and spies. A mock is an object that you pre-program with expectations. These expectations include what methods will be called, with what arguments, and what they should return. If the mock receives a call it wasn't expecting, it can raise an error.

Mocks are both **state and behavior** verifiers. You use them when you need to verify complex interactions between your system and its dependencies. The `unittest.mock.Mock` and `MagicMock` objects we've been using are, by definition, mocks.

In the previous examples, we used `patch` to create objects that we *used* as stubs or spies. The underlying object was a mock, but we chose to use only a subset of its functionality.

- When we set `return_value` and didn't make any assertions on the mock itself, we used it as a **stub**.
- When we ignored the `return_value` and only used `assert_called_once_with`, we used it as a **spy**.

A true mock-based test often does both at once.

### Summary Table

| Type  | Purpose                               | Key Question Answered                               | Example Use Case                                     |
|-------|---------------------------------------|-----------------------------------------------------|------------------------------------------------------|
| **Stub**  | Provide canned data (state)           | "Did my code produce the correct output given this input?" | Stubbing a database call to return a specific user record. |
| **Spy**   | Record interactions (behavior)        | "Did my code call the correct function with the right arguments?" | Spying on a logging function to ensure an error was logged. |
| **Mock**  | Pre-programmed expectations (state &amp; behavior) | "Did my code interact with its dependency in exactly this way?" | Mocking a payment gateway API to verify a complex transaction sequence. |

In the pytest world, you'll almost always use `unittest.mock` objects. The important thing isn't the tool itself, but how you use it. By thinking in terms of "am I testing state or behavior?", you can write clearer, more focused tests.

## Using MagicMock for Complex Scenarios

## Using MagicMock for Complex Scenarios

We've used `unittest.mock.Mock`, but it has a more capable sibling: `unittest.mock.MagicMock`. For most day-to-day testing, `MagicMock` is the tool you'll want to reach for.

So, what's "magic" about it? `MagicMock` pre-implements most of Python's "magic" methods (also called dunder methods, like `__str__`, `__len__`, `__enter__`, `__exit__`). A regular `Mock` object does not.

### The Problem with Regular Mocks

Let's see what happens when we try to use a regular `Mock` object in a situation that requires a magic method.

```python
# tests/test_magicmock_intro.py
from unittest.mock import Mock

def test_regular_mock_limitations():
    mock_obj = Mock()

    # What happens if we try to get its length?
    try:
        len(mock_obj)
    except TypeError as e:
        print(f"Calling len() on a Mock failed: {e}")

    # What happens if we use it as a context manager?
    try:
        with mock_obj as m:
            pass
    except TypeError as e:
        print(f"Using a Mock as a context manager failed: {e}")
```

Running this test would produce output like:
```
Calling len() on a Mock failed: object of type 'Mock' has no len()
Using a Mock as a context manager failed: 'Mock' object does not support the context manager protocol
```
The `Mock` object is not equipped to handle these common Python protocols out of the box. You would have to manually configure `__len__` and `__enter__`/`__exit__` methods, which is tedious.

### MagicMock to the Rescue

`MagicMock` solves this by providing default implementations for these methods. They return another `MagicMock` instance, allowing you to chain calls and make assertions naturally.

```python
# tests/test_magicmock_intro.py
from unittest.mock import MagicMock

def test_magicmock_capabilities():
    magic_mock = MagicMock()

    # It has a default length
    assert len(magic_mock) == 0

    # It can be used as a context manager
    with magic_mock as m:
        assert m is magic_mock

    # It can be iterated over (it returns an empty iterator by default)
    count = 0
    for _ in magic_mock:
        count += 1
    assert count == 0

    # You can even access items like a dictionary
    # This will create a new MagicMock on the fly!
    item = magic_mock["some_key"]
    assert isinstance(item, MagicMock)
```

`MagicMock` makes your test doubles behave more like real Python objects, reducing the friction of writing tests.

### Mocking Chained Calls

One of the most powerful features of `MagicMock` is its ability to mock chained method calls effortlessly. This is extremely common when dealing with libraries that use a fluent API (e.g., ORMs, API clients).

Consider an object that represents a connection to a web service:

```python
# src/api_client.py

class APIClient:
    def __init__(self, base_url):
        self._base_url = base_url
        # In a real client, other setup would happen here.

    @property
    def users(self):
        # This could be a different object that handles user-related endpoints.
        # We'll simulate it for the example.
        class UserEndpoint:
            def get_by_id(self, user_id):
                # This would make a real HTTP request.
                return {"id": user_id, "name": "Real User"}
        return UserEndpoint()

def get_username_from_api(client: APIClient, user_id: int) -> str:
    """Gets a user by ID and returns their name."""
    user_data = client.users.get_by_id(user_id)
    return user_data["name"]
```

To test `get_username_from_api`, we need to mock the entire chain: `client.users.get_by_id(user_id)`. With `MagicMock`, this is surprisingly simple.

```python
# tests/test_api_client.py
from unittest.mock import MagicMock
from src.api_client import get_username_from_api

def test_get_username_from_api():
    # 1. Create a MagicMock to stand in for the APIClient instance.
    mock_client = MagicMock()

    # 2. Configure the final return value at the end of the chain.
    # MagicMock automatically creates mock objects for `users` and `get_by_id`.
    mock_client.users.get_by_id.return_value = {"id": 123, "name": "Mocked Alice"}

    # 3. Call the function with the mock client.
    username = get_username_from_api(mock_client, user_id=123)

    # 4. Assert the result.
    assert username == "Mocked Alice"

    # 5. (Optional) Assert that the mock was called correctly.
    mock_client.users.get_by_id.assert_called_once_with(123)
```

This works because accessing an attribute on a `MagicMock` (like `mock_client.users`) that doesn't already exist creates a *new* `MagicMock` on the fly. You can then access attributes on *that* mock (`.get_by_id`), and so on. You only need to configure the return value of the very last call in the chain.

**Rule of thumb:** Use `MagicMock` as your default choice over `Mock`. Switch to `Mock` only if you specifically need the stricter behavior of not having magic methods implemented.

## Mocking Entire Classes

## Mocking Entire Classes

So far, we've mostly patched functions or methods on existing objects. A more powerful technique is to patch an entire class. This is essential when you want to prevent a class from being instantiated at all, controlling its creation and the behavior of its instances.

This is most common when dealing with classes that manage external resources, like database connections or network clients.

### The Scenario: A Database Connector

Let's define a simple class that connects to a database. We absolutely do not want this class's `__init__` method to run during our unit tests, as it would try to establish a real network connection.

```python
# src/database.py
import time

class DatabaseConnector:
    def __init__(self, connection_string):
        print(f"\nAttempting to connect to {connection_string}...")
        # Simulate a slow network connection
        time.sleep(2)
        self.connection_string = connection_string
        print("Connection successful!")

    def fetch_data(self, query):
        # In a real scenario, this would execute the query.
        return f"Data for query: {query}"

def get_report_for_user(user_id):
    """Connects to the DB and generates a report."""
    connector = DatabaseConnector("postgresql://user:pass@host/db")
    report_data = connector.fetch_data(f"SELECT * FROM reports WHERE user_id={user_id}")
    return f"Report: {report_data}"
```

Our goal is to test `get_report_for_user` without ever running the `DatabaseConnector.__init__` method. We can achieve this by patching the `DatabaseConnector` class itself.

### Patching a Class

When you use `patch` on a class, pytest replaces the class with a `MagicMock`. Any attempt to instantiate the class (e.g., `DatabaseConnector(...)`) will "call" this mock. The result of that call—the object that is treated as the instance—is the `return_value` of the class mock.

```python
# tests/test_database.py
from unittest.mock import patch
from src.database import get_report_for_user

def test_get_report_for_user_with_mocked_class():
    # The target string is 'module_name.ClassName'
    with patch('src.database.DatabaseConnector') as MockConnector:
        # At this point, DatabaseConnector is a MagicMock class.

        # Let's configure the *instance* that will be created.
        # MockConnector is the mock class.
        # MockConnector.return_value is the mock instance.
        mock_instance = MockConnector.return_value
        mock_instance.fetch_data.return_value = "Mocked report data"

        # Now, call the function that uses the class.
        result = get_report_for_user(user_id=42)

        # --- Assertions ---

        # 1. Assert that the class was instantiated correctly.
        MockConnector.assert_called_once_with("postgresql://user:pass@host/db")

        # 2. Assert that the method on the instance was called.
        mock_instance.fetch_data.assert_called_once_with("SELECT * FROM reports WHERE user_id=42")

        # 3. Assert the final output of our function.
        assert result == "Report: Mocked report data"
```

### Deconstructing the Magic

Let's break down the most critical and often confusing part of that test:

1.  **`patch('src.database.DatabaseConnector') as MockConnector`**:
    -   Inside the `with` block, the name `DatabaseConnector` in the `src.database` module now points to our `MagicMock`, which we've called `MockConnector`.

2.  **`connector = DatabaseConnector(...)`**:
    -   When `get_report_for_user` executes this line, it's not calling the real class constructor. It's calling our `MockConnector` mock.
    -   This is why `MockConnector.assert_called_once_with(...)` works. We are verifying the arguments passed to the "constructor".

3.  **`mock_instance = MockConnector.return_value`**:
    -   The result of calling a mock is its `return_value` attribute.
    -   Therefore, the `connector` object inside our function will be whatever `MockConnector.return_value` is. By default, it's another `MagicMock`.
    -   We grab a reference to this "instance mock" so we can configure its methods, like `fetch_data`.

4.  **`mock_instance.fetch_data.return_value = "..."`**:
    -   We are programming the `fetch_data` method on the *future instance* to return our desired string.

5.  **`connector.fetch_data(...)`**:
    -   Inside the function, this line calls the `fetch_data` method on our `mock_instance`. It returns the value we configured, and the call is recorded.
    -   This is why `mock_instance.fetch_data.assert_called_once_with(...)` works.

This pattern is fundamental for isolating your code from complex dependencies. You control class instantiation, you control the instance that's created, and you control the behavior of that instance's methods.

## Mocking Properties and Attributes

## Mocking Properties and Attributes

Sometimes you don't need to mock an entire class or method, but just a single attribute or a `@property` on an object. This is useful for testing logic that branches based on an object's state.

### Mocking Simple Attributes

Mocking a simple data attribute is straightforward using `patch.object`. This is often done to simulate different states of an object.

Consider a class that checks if a user is an administrator.

```python
# src/auth.py

class User:
    def __init__(self, username, is_admin=False):
        self.username = username
        self.is_admin = is_admin

def perform_admin_task(user: User):
    if not user.is_admin:
        raise PermissionError("User is not an admin")
    return "Admin task successful"
```

In our test, we can take a non-admin user object and temporarily make it an admin by patching the `is_admin` attribute.

```python
# tests/test_auth_attributes.py
import pytest
from unittest.mock import patch
from src.auth import User, perform_admin_task

def test_perform_admin_task_for_non_admin():
    user = User("alice")
    assert not user.is_admin
    with pytest.raises(PermissionError, match="User is not an admin"):
        perform_admin_task(user)

def test_perform_admin_task_by_patching_attribute():
    # Start with a non-admin user
    user = User("alice")

    # Use patch.object to temporarily change the is_admin attribute
    with patch.object(user, 'is_admin', True):
        # Inside this block, user.is_admin will be True
        result = perform_admin_task(user)
        assert result == "Admin task successful"

    # Outside the block, the attribute is restored to its original value
    assert not user.is_admin
```

`patch.object(target_object, 'attribute_name', new_value)` is a clean, temporary way to manipulate an object's state for the duration of a test.

### Mocking `@property` Methods

Mocking a property (a method decorated with `@property`) is slightly more complex. A property looks like an attribute but is actually a method that gets executed upon access. Simply patching it with a value won't work as intended.

Let's add a property to our `User` class.

```python
# src/auth_property.py

class UserWithProperty:
    def __init__(self, first_name, last_name):
        self.first_name = first_name
        self.last_name = last_name
        self._permissions = {"read": True, "write": False}

    @property
    def full_name(self):
        print("\nCalculating full_name...")
        return f"{self.first_name} {self.last_name}"

    def has_permission(self, permission):
        return self._permissions.get(permission, False)

def greet_user(user: UserWithProperty):
    return f"Hello, {user.full_name}!"
```

Let's say the `full_name` property is computationally expensive, and we want to avoid running it in a test for `greet_user`. We need to replace the property with a mock that returns a fixed value. For this, we use `unittest.mock.PropertyMock`.

```python
# tests/test_auth_property.py
from unittest.mock import patch, PropertyMock
from src.auth_property import UserWithProperty, greet_user

def test_greet_user_with_mocked_property():
    user = UserWithProperty("Alice", "Smith")

    # To mock a property, we patch it on the CLASS, not the instance.
    # We use `new_callable=PropertyMock` to replace the property correctly.
    with patch.object(UserWithProperty, 'full_name', new_callable=PropertyMock) as mock_full_name:
        # Configure the return value of the mocked property
        mock_full_name.return_value = "Mocked Name"

        # Now, when greet_user accesses user.full_name, it will hit our mock
        # instead of running the original property code.
        greeting = greet_user(user)

        # The original property's print statement will not be executed.
        assert greeting == "Hello, Mocked Name!"

        # We can also assert that the property was accessed
        mock_full_name.assert_called_once()
```

Let's break down the key line: `patch.object(UserWithProperty, 'full_name', new_callable=PropertyMock)`.

-   **`UserWithProperty`**: We patch the property on the class. This ensures that any instance of the class created within the `patch` context will use the mock.
-   **`'full_name'`**: The name of the property to replace.
-   **`new_callable=PropertyMock`**: This is the crucial part. It tells `patch` not to replace `full_name` with a regular `MagicMock`, but specifically with a `PropertyMock` instance. This special mock is designed to correctly emulate the behavior of a property (i.e., it can be "get", "set", or "deleted").

By using this pattern, you can effectively isolate your tests from complex or slow property logic, focusing solely on the behavior of the function you are testing.

## Testing Code That Uses External Libraries

## Testing Code That Uses External Libraries

One of the most vital uses of mocking is to isolate your application from external libraries, especially those that perform I/O operations like network requests. This makes your tests fast, reliable, and independent of external services.

The popular `requests` library for making HTTP calls is a perfect candidate for this.

### The Scenario: A GitHub API Client

Let's write a simple function that fetches the names of a user's public repositories from the GitHub API.

```python
# src/github_client.py
import requests

def get_public_repo_names(username: str):
    """
    Fetches a user's public repositories from GitHub and returns their names.
    """
    if not isinstance(username, str) or not username:
        raise ValueError("Username must be a non-empty string")

    url = f"https://api.github.com/users/{username}/repos"
    try:
        response = requests.get(url, timeout=5)
        response.raise_for_status()  # Raises an HTTPError for bad responses (4xx or 5xx)
    except requests.exceptions.RequestException as e:
        print(f"An error occurred: {e}")
        return []

    repos = response.json()
    return [repo["name"] for repo in repos]
```

### The Wrong Way: Making Real Network Calls

A naive test for this function might look like this. **Do not do this in your actual test suite.**

```python
# tests/test_github_client_bad.py
from src.github_client import get_public_repo_names

# This is a bad test!
def test_get_public_repo_names_real_call():
    # 1. Slow: This test takes seconds to run due to the network call.
    # 2. Unreliable: It will fail if there's no internet connection or if GitHub is down.
    # 3. Brittle: It will fail if the 'pytest-dev' user changes their repo names.
    repos = get_public_repo_names("pytest-dev")
    assert "pytest" in repos
```

This test violates the core principles of a good unit test. It's slow, dependent on external factors, and not deterministic.

### The Right Way: Mocking `requests.get`

The correct approach is to use `patch` to intercept the call to `requests.get`. We will replace it with a mock that returns a predictable, simulated response. This allows us to test our function's logic in complete isolation.

```python
# tests/test_github_client_good.py
import pytest
import requests
from unittest.mock import patch, MagicMock
from src.github_client import get_public_repo_names

@patch('src.github_client.requests.get')
def test_get_public_repo_names_success(mock_get):
    """Test the success case where the API returns a list of repos."""
    # 1. Arrange: Configure the mock to simulate a successful API response.
    # The return_value of requests.get should be a mock response object.
    mock_response = MagicMock()
    mock_response.status_code = 200
    mock_response.json.return_value = [
        {"name": "pytest"},
        {"name": "pytest-xdist"},
        {"name": "pytest-cov"},
    ]
    mock_get.return_value = mock_response

    # 2. Act: Call our function.
    repo_names = get_public_repo_names("pytest-dev")

    # 3. Assert: Verify our function's behavior.
    # Check that requests.get was called with the correct URL.
    mock_get.assert_called_once_with("https://api.github.com/users/pytest-dev/repos", timeout=5)

    # Check that our function correctly processed the mock JSON.
    assert repo_names == ["pytest", "pytest-xdist", "pytest-cov"]

@patch('src.github_client.requests.get')
def test_get_public_repo_names_http_error(mock_get):
    """Test the case where the API returns an error status code."""
    # 1. Arrange: Configure the mock to simulate an HTTP error.
    mock_response = MagicMock()
    # The raise_for_status method should raise an error.
    mock_response.raise_for_status.side_effect = requests.exceptions.HTTPError("404 Not Found")
    mock_get.return_value = mock_response

    # 2. Act: Call our function.
    repo_names = get_public_repo_names("nonexistent-user")

    # 3. Assert: Verify our function handles the error gracefully.
    mock_get.assert_called_once_with("https://api.github.com/users/nonexistent-user/repos", timeout=5)
    assert repo_names == []

def test_get_public_repo_names_invalid_input():
    """Test the input validation logic without any mocking."""
    with pytest.raises(ValueError, match="Username must be a non-empty string"):
        get_public_repo_names("")
```

### Analysis of the Good Tests

1.  **`@patch('src.github_client.requests.get')`**: We patch `requests.get` *where it is used* (`src.github_client`), not where it is defined (`requests`). This is a critical rule of patching.
2.  **Configuring the Mock Response**: We create a `MagicMock` to act as the `response` object. We then configure its attributes and methods (`.status_code`, `.json()`, `.raise_for_status()`) to match what our code expects from a real `requests.Response` object.
3.  **Testing Different Scenarios**: We can now easily test various scenarios without relying on the real GitHub API. We can simulate success, HTTP errors, network timeouts (`side_effect=requests.exceptions.Timeout`), and more.
4.  **Fast and Reliable**: These tests run in milliseconds and will produce the same result every single time, regardless of network conditions.

This pattern is applicable to any external library. Identify the boundary between your code and the library, and place your mock at that boundary.

## Avoiding Over-Mocking

## Avoiding Over-Mocking

Mocking is a powerful tool, but like any tool, it can be misused. **Over-mocking** is a common pitfall where tests become so tightly coupled to the implementation details of the code that they become brittle and difficult to maintain. An over-mocked test verifies *how* the code works, not *what* it achieves.

A good unit test should be a test of **behavior**, not implementation. It should validate the public contract of a function or class: given these inputs, I expect these outputs or side effects. It shouldn't care about the internal functions that were called to produce that result.

### The Smell of Over-Mocking

You might be over-mocking if your tests:
-   Break frequently when you refactor the internal implementation of a function, even though its external behavior hasn't changed.
-   Involve mocking multiple functions from your own application code within a single test.
-   Require complex setup with many `patch` decorators or context managers.
-   Test the "chain of command" (function A calls B, which calls C) rather than the final outcome.

### An Example of Over-Mocking

Let's consider a system that processes a new user registration.

```python
# src/registration.py

def validate_email(email):
    """Validates email format."""
    return "@" in email and "." in email

def create_user_record(username, email):
    """Creates a user record dictionary."""
    return {"username": username, "email": email, "status": "pending"}

def send_welcome_email(email):
    """Sends a welcome email (external service)."""
    print(f"Sending welcome email to {email}")
    # Imagine an external API call here
    return True

def register_user(username, email):
    """Main registration function."""
    if not validate_email(email):
        raise ValueError("Invalid email format")

    user_record = create_user_record(username, email)
    # In a real app, we'd save this record to a database.
    print(f"User record created: {user_record}")

    send_welcome_email(email)

    return user_record
```

Now, here is an **over-mocked test** for `register_user`.

```python
# tests/test_registration_bad.py
from unittest.mock import patch
from src.registration import register_user

# THIS IS A BAD, BRITTLE TEST
@patch('src.registration.send_welcome_email')
@patch('src.registration.create_user_record')
@patch('src.registration.validate_email')
def test_register_user_over_mocked(mock_validate, mock_create, mock_send):
    # Arrange: Mock every internal function call
    mock_validate.return_value = True
    mock_create.return_value = {"username": "testuser", "email": "test@example.com"}

    # Act
    register_user("testuser", "test@example.com")

    # Assert: Verify that every single internal function was called as expected
    mock_validate.assert_called_once_with("test@example.com")
    mock_create.assert_called_once_with("testuser", "test@example.com")
    mock_send.assert_called_once_with("test@example.com")
```

**Why is this test bad?**

Imagine we refactor `register_user` to combine `validate_email` and `create_user_record` into a single helper function. The external behavior of `register_user` is identical, but the test above would break because `validate_email` and `create_user_record` are no longer called directly. The test is coupled to the implementation, not the behavior.

### A Better Approach: Mocking at the Boundaries

A better test mocks only the true external dependencies (the "boundaries" of your system) and tests the actual outcome. In our example, the only external dependency is `send_welcome_email`. The other functions are just implementation details of our own application.

```python
# tests/test_registration_good.py
import pytest
from unittest.mock import patch
from src.registration import register_user

# THIS IS A GOOD, RESILIENT TEST
@patch('src.registration.send_welcome_email')
def test_register_user_boundary_mock(mock_send_email):
    # Arrange: We only mock the external dependency.
    username = "testuser"
    email = "test@example.com"

    # Act: Call the function and let its internal logic run.
    result = register_user(username, email)

    # Assert: Verify the public contract and the external interaction.
    # 1. Did it return the correct data structure?
    assert result == {"username": username, "email": email, "status": "pending"}

    # 2. Did it interact with the external service correctly?
    mock_send_email.assert_called_once_with(email)

def test_register_user_invalid_email():
    # Test the validation behavior directly. No mocking needed.
    with pytest.raises(ValueError, match="Invalid email format"):
        register_user("testuser", "invalid-email")
```

This improved test is far more robust. We can now refactor the internals of `register_user` as much as we want. As long as it still returns the correct user record and calls the email service, the test will pass. We are testing the *what*, not the *how*.

### Guidelines for Healthy Mocking

1.  **Mock External Systems**: Your primary targets for mocking should be systems outside your control: databases, external APIs, the file system, the clock (`datetime.now`), etc.
2.  **Don't Mock Your Own Code (Usually)**: Avoid mocking functions and classes that are part of the same application you're testing. If a helper function is complex, it should have its own dedicated unit tests. The function that calls it should use the real helper in an integration-style unit test.
3.  **Test the Public Interface**: Focus your tests on the public methods of your classes. Let the private methods be exercised via the public ones.
4.  **One Mock Per Test is a Good Goal**: If your test requires patching more than one or two things, it might be a "code smell" indicating that your function is doing too much and should be broken up (Single Responsibility Principle).

Mocking is about creating seams that let you isolate the unit of code you're testing. Use it to cut ties with the outside world, not to dissect the internal organs of your own application.
