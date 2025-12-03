# Chapter 14: Testing Web Applications

## Testing Flask Applications

## Testing Flask Applications

Web applications present unique testing challenges: they handle HTTP requests, manage sessions, interact with databases, and coordinate multiple layers of logic. In this chapter, we'll build a complete testing strategy for web applications, starting with Flask—a lightweight framework that makes the testing fundamentals crystal clear.

### The Reference Application: A Task Management API

We'll test a realistic Flask application throughout this chapter—a task management API that handles user authentication, CRUD operations, and data persistence. This will be our anchor example, progressively refined as we encounter real testing challenges.

Here's our initial Flask application:

```python
# app.py
from flask import Flask, request, jsonify
from datetime import datetime

app = Flask(__name__)

# In-memory storage (we'll address database testing later)
tasks = {}
task_id_counter = 1

@app.route('/tasks', methods=['POST'])
def create_task():
    """Create a new task"""
    global task_id_counter
    
    data = request.get_json()
    
    if not data or 'title' not in data:
        return jsonify({'error': 'Title is required'}), 400
    
    task = {
        'id': task_id_counter,
        'title': data['title'],
        'description': data.get('description', ''),
        'completed': False,
        'created_at': datetime.utcnow().isoformat()
    }
    
    tasks[task_id_counter] = task
    task_id_counter += 1
    
    return jsonify(task), 201

@app.route('/tasks/<int:task_id>', methods=['GET'])
def get_task(task_id):
    """Retrieve a specific task"""
    task = tasks.get(task_id)
    
    if not task:
        return jsonify({'error': 'Task not found'}), 404
    
    return jsonify(task), 200

@app.route('/tasks', methods=['GET'])
def list_tasks():
    """List all tasks"""
    return jsonify(list(tasks.values())), 200

if __name__ == '__main__':
    app.run(debug=True)
```

### Our First Test: The Naive Approach

Let's write our first test the way many developers initially approach Flask testing—by actually running the server:

```python
# test_app_naive.py
import requests
import subprocess
import time
import pytest

def test_create_task_naive():
    """Attempt to test by running the actual server"""
    # Start the Flask server in a subprocess
    server = subprocess.Popen(['python', 'app.py'])
    time.sleep(2)  # Wait for server to start
    
    try:
        # Make a real HTTP request
        response = requests.post(
            'http://localhost:5000/tasks',
            json={'title': 'Test task'}
        )
        
        assert response.status_code == 201
        data = response.json()
        assert data['title'] == 'Test task'
        assert 'id' in data
    finally:
        # Clean up
        server.terminate()
        server.wait()
```

Let's run this test and observe what happens:

```bash
$ pytest test_app_naive.py -v
```

**The complete output**:

```
============================= test session starts ==============================
collected 1 item

test_app_naive.py::test_create_task_naive FAILED                         [100%]

=================================== FAILURES ===================================
______________________________ test_create_task_naive __________________________

    def test_create_task_naive():
        """Attempt to test by running the actual server"""
        server = subprocess.Popen(['python', 'app.py'])
        time.sleep(2)
>       
        try:
            response = requests.post(
                'http://localhost:5000/tasks',
                json={'title': 'Test task'}
            )
E           requests.exceptions.ConnectionError: HTTPConnectionPool(host='localhost', port=5000): 
E           Max retries exceeded with url: /tasks (Caused by NewConnectionError(
E           '<urllib3.connection.HTTPConnection object at 0x7f8b8c0a3d90>: 
E           Failed to establish a new connection: [Errno 111] Connection refused'))

test_app_naive.py:12: ConnectionError
=========================== 1 failed in 2.34s ==================================
```

### Diagnostic Analysis: Reading the Failure

**The complete output** shows a `ConnectionError`, but let's parse this systematically:

**1. The summary line**: `test_app_naive.py::test_create_task_naive FAILED`
   - What this tells us: The test failed during execution, not during collection

**2. The traceback**:
```
    response = requests.post(
        'http://localhost:5000/tasks',
        json={'title': 'Test task'}
    )
E   requests.exceptions.ConnectionError: ... Connection refused
```
   - What this tells us: The HTTP request itself failed—we never even got to test our application logic
   - Key line: `Connection refused` means no server was listening on port 5000

**3. The timing**: `failed in 2.34s`
   - What this tells us: We wasted 2 seconds waiting for a server that never started properly

**Root cause identified**: Starting a real server in a test is unreliable, slow, and creates race conditions.

**Why the current approach can't solve this**: Even if we increase the sleep time or add retry logic, we're fighting against the fundamental problem—we're testing at the wrong level of abstraction.

**What we need**: A way to test Flask applications without starting an actual HTTP server.

### Iteration 1: Flask's Test Client

Flask provides a built-in test client that simulates HTTP requests without running a server. This is the standard approach for Flask testing.

```python
# test_app.py
import pytest
from app import app

def test_create_task():
    """Test task creation using Flask's test client"""
    # Create a test client
    client = app.test_client()
    
    # Make a request (no server needed!)
    response = client.post(
        '/tasks',
        json={'title': 'Write tests'}
    )
    
    # Verify the response
    assert response.status_code == 201
    data = response.get_json()
    assert data['title'] == 'Write tests'
    assert data['id'] == 1
    assert data['completed'] is False
```

Let's run this improved version:

```bash
$ pytest test_app.py::test_create_task -v
```

**Output**:
```
============================= test session starts ==============================
collected 1 item

test_app.py::test_create_task PASSED                                     [100%]

============================== 1 passed in 0.03s ===============================
```

**Expected vs. Actual improvement**:
- **Speed**: 2.34s → 0.03s (78x faster)
- **Reliability**: Connection errors eliminated
- **Simplicity**: No subprocess management, no sleep calls

This works! But let's add another test to verify retrieval:

```python
# test_app.py (continued)
def test_get_task():
    """Test retrieving a task"""
    client = app.test_client()
    
    # First, create a task
    create_response = client.post(
        '/tasks',
        json={'title': 'Test task'}
    )
    task_id = create_response.get_json()['id']
    
    # Now retrieve it
    response = client.get(f'/tasks/{task_id}')
    
    assert response.status_code == 200
    data = response.get_json()
    assert data['title'] == 'Test task'
```

Run both tests:

```bash
$ pytest test_app.py -v
```

**The complete output**:
```
============================= test session starts ==============================
collected 2 items

test_app.py::test_create_task PASSED                                     [ 50%]
test_app.py::test_get_task FAILED                                        [100%]

=================================== FAILURES ===================================
_________________________________ test_get_task ________________________________

    def test_get_task():
        """Test retrieving a task"""
        client = app.test_client()
        
        create_response = client.post(
            '/tasks',
            json={'title': 'Test task'}
        )
        task_id = create_response.get_json()['id']
>       
        response = client.get(f'/tasks/{task_id}')
        
        assert response.status_code == 200
        data = response.get_json()
        assert data['title'] == 'Test task'
E       AssertionError: assert 'Test task' == 'Write tests'
E         - Write tests
E         + Test task

test_app.py:25: AssertionError
=========================== 1 failed, 1 passed in 0.05s =======================
```

### Diagnostic Analysis: Test Pollution

**Let's parse this section by section**:

**1. The summary line**: `test_get_task FAILED`
   - What this tells us: The second test failed, but the first passed

**2. The assertion introspection**:
```
E       AssertionError: assert 'Test task' == 'Write tests'
E         - Write tests
E         + Test task
```
   - What this tells us: We expected to retrieve "Test task" but got "Write tests"
   - Key insight: "Write tests" was the title from the FIRST test

**3. The pattern**: First test passes, second test fails
   - What this tells us: Tests are interfering with each other

**Root cause identified**: Our Flask app uses global state (`tasks` dictionary). The first test creates a task with ID 1, and that task persists into the second test. When the second test creates its task, it gets ID 2, but then tries to retrieve ID 1—which contains data from the first test.

**Why the current approach can't solve this**: Simply reordering tests or running them individually would mask the problem, not fix it.

**What we need**: A way to reset the application state between tests.

### Iteration 2: Application State Management with Fixtures

The solution is to create a fresh application instance for each test using fixtures:

```python
# test_app.py (refactored)
import pytest
from app import app as flask_app

@pytest.fixture
def app():
    """Create a fresh Flask app for each test"""
    # Configure the app for testing
    flask_app.config['TESTING'] = True
    
    yield flask_app
    
    # Clean up: reset global state
    from app import tasks, task_id_counter
    tasks.clear()
    # Note: We can't reset task_id_counter because it's not mutable
    # We'll address this limitation next

@pytest.fixture
def client(app):
    """Create a test client"""
    return app.test_client()

def test_create_task(client):
    """Test task creation"""
    response = client.post(
        '/tasks',
        json={'title': 'Write tests'}
    )
    
    assert response.status_code == 201
    data = response.get_json()
    assert data['title'] == 'Write tests'
    assert data['completed'] is False

def test_get_task(client):
    """Test retrieving a task"""
    # Create a task
    create_response = client.post(
        '/tasks',
        json={'title': 'Test task'}
    )
    task_id = create_response.get_json()['id']
    
    # Retrieve it
    response = client.get(f'/tasks/{task_id}')
    
    assert response.status_code == 200
    data = response.get_json()
    assert data['title'] == 'Test task'
```

Run the tests again:

```bash
$ pytest test_app.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 2 items

test_app.py::test_create_task PASSED                                     [ 50%]
test_app.py::test_get_task PASSED                                        [100%]

============================== 2 passed in 0.04s ===============================
```

**Expected vs. Actual improvement**:
- Tests now pass consistently regardless of execution order
- Each test gets a clean slate
- The fixture pattern makes the setup explicit and reusable

**Limitation preview**: We're clearing the `tasks` dictionary, but we can't reset `task_id_counter` because it's a module-level integer. This means task IDs will continue incrementing across tests. Let's see why this matters:

```python
# test_app.py (add this test)
def test_task_ids_are_sequential(client):
    """Verify that task IDs start at 1"""
    response1 = client.post('/tasks', json={'title': 'First'})
    response2 = client.post('/tasks', json={'title': 'Second'})
    
    assert response1.get_json()['id'] == 1
    assert response2.get_json()['id'] == 2
```

Run all tests:

```bash
$ pytest test_app.py -v
```

**The complete output**:
```
============================= test session starts ==============================
collected 3 items

test_app.py::test_create_task PASSED                                     [ 33%]
test_app.py::test_get_task PASSED                                        [ 66%]
test_app.py::test_task_ids_are_sequential FAILED                         [100%]

=================================== FAILURES ===================================
_________________________ test_task_ids_are_sequential _________________________

    def test_task_ids_are_sequential(client):
        """Verify that task IDs start at 1"""
        response1 = client.post('/tasks', json={'title': 'First'})
        response2 = client.post('/tasks', json={'title': 'Second'})
        
>       assert response1.get_json()['id'] == 1
E       AssertionError: assert 3 == 1
E        +  where 3 = {'id': 3, 'title': 'First', 'description': '', 'completed': False, ...}['id']
E        +    where {'id': 3, 'title': 'First', 'description': '', 'completed': False, ...} = <bound method Response.get_json of <Response 201 CREATED>>()

test_app.py:58: AssertionError
=========================== 1 failed, 2 passed in 0.05s =======================
```

### Diagnostic Analysis: Incomplete State Reset

**Let's parse this section by section**:

**1. The assertion introspection**:
```
E       AssertionError: assert 3 == 1
E        +  where 3 = {'id': 3, 'title': 'First', ...}['id']
```
   - What this tells us: The first task created in this test got ID 3, not ID 1
   - Key insight: IDs 1 and 2 were used by previous tests

**2. The pattern**: The test would pass if run in isolation
   - What this tells us: This is a test order dependency issue

**Root cause identified**: We're clearing the `tasks` dictionary but not resetting `task_id_counter`. The counter keeps incrementing across tests.

**Why the current approach can't solve this**: Python integers are immutable. We can't modify `task_id_counter` from outside the module without restructuring the application code.

**What we need**: Either refactor the application to make state resettable, or adjust our testing strategy to not depend on specific ID values.

### Iteration 3: Application Factory Pattern

The professional solution is to refactor the Flask app using the **application factory pattern**. This makes the app testable by design:

```python
# app.py (refactored)
from flask import Flask, request, jsonify
from datetime import datetime

def create_app(config=None):
    """Application factory function"""
    app = Flask(__name__)
    
    if config:
        app.config.update(config)
    
    # Application state stored on the app object
    app.tasks = {}
    app.task_id_counter = 1
    
    @app.route('/tasks', methods=['POST'])
    def create_task():
        """Create a new task"""
        data = request.get_json()
        
        if not data or 'title' not in data:
            return jsonify({'error': 'Title is required'}), 400
        
        task = {
            'id': app.task_id_counter,
            'title': data['title'],
            'description': data.get('description', ''),
            'completed': False,
            'created_at': datetime.utcnow().isoformat()
        }
        
        app.tasks[app.task_id_counter] = task
        app.task_id_counter += 1
        
        return jsonify(task), 201
    
    @app.route('/tasks/<int:task_id>', methods=['GET'])
    def get_task(task_id):
        """Retrieve a specific task"""
        task = app.tasks.get(task_id)
        
        if not task:
            return jsonify({'error': 'Task not found'}), 404
        
        return jsonify(task), 200
    
    @app.route('/tasks', methods=['GET'])
    def list_tasks():
        """List all tasks"""
        return jsonify(list(app.tasks.values())), 200
    
    return app

if __name__ == '__main__':
    app = create_app()
    app.run(debug=True)
```

Now our test fixtures become much cleaner:

```python
# test_app.py (final version)
import pytest
from app import create_app

@pytest.fixture
def app():
    """Create a fresh Flask app for each test"""
    app = create_app({'TESTING': True})
    yield app
    # No cleanup needed—each test gets a new app instance

@pytest.fixture
def client(app):
    """Create a test client"""
    return app.test_client()

def test_create_task(client):
    """Test task creation"""
    response = client.post(
        '/tasks',
        json={'title': 'Write tests'}
    )
    
    assert response.status_code == 201
    data = response.get_json()
    assert data['title'] == 'Write tests'
    assert data['completed'] is False

def test_get_task(client):
    """Test retrieving a task"""
    create_response = client.post(
        '/tasks',
        json={'title': 'Test task'}
    )
    task_id = create_response.get_json()['id']
    
    response = client.get(f'/tasks/{task_id}')
    
    assert response.status_code == 200
    data = response.get_json()
    assert data['title'] == 'Test task'

def test_task_ids_are_sequential(client):
    """Verify that task IDs start at 1"""
    response1 = client.post('/tasks', json={'title': 'First'})
    response2 = client.post('/tasks', json={'title': 'Second'})
    
    assert response1.get_json()['id'] == 1
    assert response2.get_json()['id'] == 2

def test_get_nonexistent_task(client):
    """Test 404 handling"""
    response = client.get('/tasks/999')
    
    assert response.status_code == 404
    assert 'error' in response.get_json()

def test_create_task_missing_title(client):
    """Test validation"""
    response = client.post('/tasks', json={})
    
    assert response.status_code == 400
    assert 'error' in response.get_json()
```

Run all tests:

```bash
$ pytest test_app.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 5 items

test_app.py::test_create_task PASSED                                     [ 20%]
test_app.py::test_get_task PASSED                                        [ 40%]
test_app.py::test_task_ids_are_sequential PASSED                         [ 60%]
test_app.py::test_get_nonexistent_task PASSED                            [ 80%]
test_app.py::test_create_task_missing_title PASSED                       [100%]

============================== 5 passed in 0.06s ===============================
```

**Expected vs. Actual improvement**:
- All tests pass consistently
- No manual cleanup required
- Each test gets completely isolated state
- Application code is now more maintainable (factory pattern is a best practice)

### Common Failure Modes and Their Signatures

#### Symptom: Tests pass individually but fail when run together

**Pytest output pattern**:
```
test_app.py::test_first PASSED
test_app.py::test_second FAILED
```

**Diagnostic clues**:
- Tests pass when run with `-k test_second` (isolated)
- Failure involves unexpected data from previous tests
- Assertion shows values from earlier test cases

**Root cause**: Shared mutable state between tests (global variables, class attributes, module-level collections)

**Solution**: Use application factory pattern + fixtures to create fresh instances per test

#### Symptom: ConnectionError or "Address already in use"

**Pytest output pattern**:
```
E   requests.exceptions.ConnectionError: ... Connection refused
```
or
```
E   OSError: [Errno 48] Address already in use
```

**Diagnostic clues**:
- Error occurs during test setup or first request
- Involves `subprocess`, `threading`, or actual server startup
- Tests are slow (>1 second each)

**Root cause**: Attempting to run a real HTTP server during tests

**Solution**: Use Flask's `test_client()` instead of starting an actual server

#### Symptom: Tests fail with "Working outside of application context"

**Pytest output pattern**:
```
E   RuntimeError: Working outside of application context.
```

**Diagnostic clues**:
- Error occurs when accessing Flask features (current_app, g, session)
- Happens in helper functions or fixtures
- Code works fine when running the actual application

**Root cause**: Flask requires an application context for certain operations

**Solution**: Use `app.app_context()` or `app.test_request_context()` in fixtures (we'll cover this in section 14.4)

### When to Apply This Solution

**What it optimizes for**:
- Test isolation and reliability
- Fast test execution (no network overhead)
- Ability to test edge cases and error conditions

**What it sacrifices**:
- Doesn't test actual HTTP server behavior
- Doesn't catch deployment configuration issues
- Doesn't test reverse proxy or load balancer interactions

**When to choose this approach**:
- Unit and integration testing of application logic
- Testing request handlers, validation, and business logic
- CI/CD pipelines where speed matters
- Development workflow (fast feedback loop)

**When to avoid this approach**:
- End-to-end testing of deployed applications
- Testing server configuration or deployment issues
- Load testing or performance testing
- Testing interactions with external services in production-like environments

**Code characteristics**:
- **Setup complexity**: Low (just create fixtures)
- **Maintenance burden**: Low (tests are simple and fast)
- **Testability**: High (easy to test edge cases and error conditions)

## Testing Django Applications

## Testing Django Applications

Django takes a different approach to testing than Flask. While Flask is minimalist and requires you to build your own patterns, Django comes with a comprehensive testing framework built in. Let's explore how to test Django applications effectively using pytest.

### The Reference Application: A Blog API

We'll build and test a Django REST API for a blog system. This will demonstrate Django-specific testing patterns while building on the Flask concepts we just learned.

First, let's set up our Django models and views:

```python
# blog/models.py
from django.db import models
from django.contrib.auth.models import User

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    published = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        ordering = ['-created_at']
    
    def __str__(self):
        return self.title

class Comment(models.Model):
    post = models.ForeignKey(Post, on_delete=models.CASCADE, related_name='comments')
    author = models.ForeignKey(User, on_delete=models.CASCADE)
    content = models.TextField()
    created_at = models.DateTimeField(auto_now_add=True)
    
    class Meta:
        ordering = ['created_at']
    
    def __str__(self):
        return f'Comment by {self.author.username} on {self.post.title}'
```

```python
# blog/views.py
from django.shortcuts import get_object_or_404
from django.http import JsonResponse
from django.views.decorators.http import require_http_methods
from django.views.decorators.csrf import csrf_exempt
import json
from .models import Post, Comment

@csrf_exempt
@require_http_methods(["POST"])
def create_post(request):
    """Create a new blog post"""
    try:
        data = json.loads(request.body)
    except json.JSONDecodeError:
        return JsonResponse({'error': 'Invalid JSON'}, status=400)
    
    if not data.get('title'):
        return JsonResponse({'error': 'Title is required'}, status=400)
    
    if not request.user.is_authenticated:
        return JsonResponse({'error': 'Authentication required'}, status=401)
    
    post = Post.objects.create(
        title=data['title'],
        content=data.get('content', ''),
        author=request.user,
        published=data.get('published', False)
    )
    
    return JsonResponse({
        'id': post.id,
        'title': post.title,
        'content': post.content,
        'author': post.author.username,
        'published': post.published,
        'created_at': post.created_at.isoformat()
    }, status=201)

@require_http_methods(["GET"])
def get_post(request, post_id):
    """Retrieve a specific post"""
    post = get_object_or_404(Post, id=post_id)
    
    return JsonResponse({
        'id': post.id,
        'title': post.title,
        'content': post.content,
        'author': post.author.username,
        'published': post.published,
        'created_at': post.created_at.isoformat()
    })

@require_http_methods(["GET"])
def list_posts(request):
    """List all published posts"""
    posts = Post.objects.filter(published=True)
    
    return JsonResponse({
        'posts': [
            {
                'id': post.id,
                'title': post.title,
                'author': post.author.username,
                'created_at': post.created_at.isoformat()
            }
            for post in posts
        ]
    })
```

### Iteration 1: The Django TestCase Approach

Django provides `django.test.TestCase`, which many developers use. Let's start there to understand why pytest offers advantages:

```python
# blog/tests.py (Django's built-in approach)
from django.test import TestCase
from django.contrib.auth.models import User
from .models import Post

class PostTestCase(TestCase):
    def setUp(self):
        """Run before each test"""
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
    
    def test_create_post(self):
        """Test post creation"""
        post = Post.objects.create(
            title='Test Post',
            content='Test content',
            author=self.user,
            published=True
        )
        
        self.assertEqual(post.title, 'Test Post')
        self.assertEqual(post.author, self.user)
        self.assertTrue(post.published)
    
    def test_post_str_representation(self):
        """Test string representation"""
        post = Post.objects.create(
            title='Test Post',
            content='Test content',
            author=self.user
        )
        
        self.assertEqual(str(post), 'Test Post')
```

Run this with Django's test runner:

```bash
$ python manage.py test blog
```

**Output**:
```
Creating test database for alias 'default'...
System check identified no issues (0 silenced).
..
----------------------------------------------------------------------
Ran 2 tests in 0.123s

OK
Destroying test database for alias 'default'...
```

This works, but notice several limitations:

1. **Verbose syntax**: `self.assertEqual()` instead of simple `assert`
2. **Class-based structure**: Must inherit from `TestCase`
3. **setUp/tearDown pattern**: Less flexible than pytest fixtures
4. **Limited introspection**: Failures show less detail than pytest

### Iteration 2: Switching to Pytest with pytest-django

Let's refactor to use pytest with the `pytest-django` plugin:

```bash
$ pip install pytest-django
```

Configure pytest for Django:

```ini
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = myproject.settings
python_files = tests.py test_*.py *_tests.py
```

Now rewrite our tests using pytest:

```python
# blog/test_models.py
import pytest
from django.contrib.auth.models import User
from blog.models import Post

@pytest.fixture
def user(db):
    """Create a test user"""
    return User.objects.create_user(
        username='testuser',
        password='testpass123'
    )

@pytest.mark.django_db
def test_create_post(user):
    """Test post creation"""
    post = Post.objects.create(
        title='Test Post',
        content='Test content',
        author=user,
        published=True
    )
    
    assert post.title == 'Test Post'
    assert post.author == user
    assert post.published is True

@pytest.mark.django_db
def test_post_str_representation(user):
    """Test string representation"""
    post = Post.objects.create(
        title='Test Post',
        content='Test content',
        author=user
    )
    
    assert str(post) == 'Test Post'
```

Run with pytest:

```bash
$ pytest blog/test_models.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 2 items

blog/test_models.py::test_create_post PASSED                             [ 50%]
blog/test_models.py::test_post_str_representation PASSED                 [100%]

============================== 2 passed in 0.18s ===============================
```

**Expected vs. Actual improvement**:
- **Cleaner syntax**: `assert` instead of `self.assertEqual()`
- **Better failures**: Pytest's introspection shows exactly what differed
- **Fixture composition**: Can combine fixtures easily
- **Parametrization**: Can use `@pytest.mark.parametrize` (not available in Django TestCase)

**Limitation preview**: We're testing models, but we haven't tested the views yet. Let's see what happens when we try to test the API endpoints:

```python
# blog/test_views.py
import pytest
from django.contrib.auth.models import User
from blog.models import Post

@pytest.fixture
def user(db):
    """Create a test user"""
    return User.objects.create_user(
        username='testuser',
        password='testpass123'
    )

@pytest.mark.django_db
def test_create_post_view(user):
    """Test the create_post view"""
    from django.test import Client
    
    client = Client()
    
    response = client.post(
        '/blog/posts/',
        data={'title': 'Test Post', 'content': 'Test content'},
        content_type='application/json'
    )
    
    assert response.status_code == 201
```

Run this test:

```bash
$ pytest blog/test_views.py::test_create_post_view -v
```

**The complete output**:
```
============================= test session starts ==============================
collected 1 item

blog/test_views.py::test_create_post_view FAILED                         [100%]

=================================== FAILURES ===================================
__________________________ test_create_post_view _______________________________

user = <User: testuser>

    @pytest.mark.django_db
    def test_create_post_view(user):
        """Test the create_post view"""
        from django.test import Client
        
        client = Client()
        
        response = client.post(
            '/blog/posts/',
            data={'title': 'Test Post', 'content': 'Test content'},
            content_type='application/json'
        )
        
>       assert response.status_code == 201
E       AssertionError: assert 401 == 201
E        +  where 401 = <JsonResponse status_code=401, "application/json">.status_code

blog/test_views.py:18: AssertionError
=========================== 1 failed in 0.21s =======================
```

### Diagnostic Analysis: Authentication Required

**Let's parse this section by section**:

**1. The assertion introspection**:
```
E       AssertionError: assert 401 == 201
E        +  where 401 = <JsonResponse status_code=401, ...>.status_code
```
   - What this tells us: We got a 401 (Unauthorized) instead of 201 (Created)
   - Key insight: The view requires authentication, but our test client isn't authenticated

**2. Looking at the view code**:
```python
if not request.user.is_authenticated:
    return JsonResponse({'error': 'Authentication required'}, status=401)
```
   - What this tells us: The view explicitly checks for authentication

**Root cause identified**: We created a user fixture but didn't authenticate the test client with that user.

**Why the current approach can't solve this**: Simply creating a user doesn't automatically authenticate requests.

**What we need**: A way to authenticate the test client as our test user.

### Iteration 3: Authenticated Test Client

Django's test client provides a `force_login()` method for testing authenticated views:

```python
# blog/test_views.py (refactored)
import pytest
import json
from django.contrib.auth.models import User
from django.test import Client
from blog.models import Post

@pytest.fixture
def user(db):
    """Create a test user"""
    return User.objects.create_user(
        username='testuser',
        password='testpass123'
    )

@pytest.fixture
def authenticated_client(user):
    """Create an authenticated test client"""
    client = Client()
    client.force_login(user)
    return client

@pytest.mark.django_db
def test_create_post_view(authenticated_client):
    """Test the create_post view"""
    response = authenticated_client.post(
        '/blog/posts/',
        data=json.dumps({'title': 'Test Post', 'content': 'Test content'}),
        content_type='application/json'
    )
    
    assert response.status_code == 201
    data = response.json()
    assert data['title'] == 'Test Post'
    assert data['content'] == 'Test content'

@pytest.mark.django_db
def test_create_post_requires_authentication():
    """Test that unauthenticated requests are rejected"""
    client = Client()
    
    response = client.post(
        '/blog/posts/',
        data=json.dumps({'title': 'Test Post'}),
        content_type='application/json'
    )
    
    assert response.status_code == 401
    assert 'error' in response.json()

@pytest.mark.django_db
def test_create_post_requires_title(authenticated_client):
    """Test validation"""
    response = authenticated_client.post(
        '/blog/posts/',
        data=json.dumps({}),
        content_type='application/json'
    )
    
    assert response.status_code == 400
    assert 'error' in response.json()

@pytest.mark.django_db
def test_get_post(authenticated_client, user):
    """Test retrieving a post"""
    # Create a post directly in the database
    post = Post.objects.create(
        title='Test Post',
        content='Test content',
        author=user,
        published=True
    )
    
    response = authenticated_client.get(f'/blog/posts/{post.id}/')
    
    assert response.status_code == 200
    data = response.json()
    assert data['title'] == 'Test Post'
    assert data['author'] == 'testuser'

@pytest.mark.django_db
def test_list_posts_shows_only_published(authenticated_client, user):
    """Test that only published posts appear in the list"""
    # Create published and unpublished posts
    Post.objects.create(
        title='Published Post',
        content='Content',
        author=user,
        published=True
    )
    Post.objects.create(
        title='Draft Post',
        content='Content',
        author=user,
        published=False
    )
    
    response = authenticated_client.get('/blog/posts/')
    
    assert response.status_code == 200
    data = response.json()
    assert len(data['posts']) == 1
    assert data['posts'][0]['title'] == 'Published Post'
```

Run all tests:

```bash
$ pytest blog/test_views.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 5 items

blog/test_views.py::test_create_post_view PASSED                         [ 20%]
blog/test_views.py::test_create_post_requires_authentication PASSED      [ 40%]
blog/test_views.py::test_create_post_requires_title PASSED               [ 60%]
blog/test_views.py::test_get_post PASSED                                 [ 80%]
blog/test_views.py::test_list_posts_shows_only_published PASSED          [100%]

============================== 5 passed in 0.34s ===============================
```

**Expected vs. Actual improvement**:
- All tests pass with proper authentication
- Fixture composition makes it easy to create authenticated clients
- Tests are clear about what they're testing (authentication, validation, filtering)

### Using pytest-django's Built-in Fixtures

pytest-django provides several useful fixtures out of the box:

```python
# blog/test_views_advanced.py
import pytest
import json
from blog.models import Post

@pytest.fixture
def user(django_user_model):
    """Create a test user using django_user_model fixture"""
    return django_user_model.objects.create_user(
        username='testuser',
        password='testpass123'
    )

@pytest.mark.django_db
def test_create_post_with_client_fixture(client, user):
    """Test using pytest-django's client fixture"""
    client.force_login(user)
    
    response = client.post(
        '/blog/posts/',
        data=json.dumps({'title': 'Test Post'}),
        content_type='application/json'
    )
    
    assert response.status_code == 201

@pytest.mark.django_db
def test_database_access_with_transactional_db(transactional_db, user):
    """Test with transactional database access"""
    # transactional_db allows testing transaction behavior
    post = Post.objects.create(
        title='Test Post',
        content='Content',
        author=user
    )
    
    assert Post.objects.count() == 1

def test_settings_override(settings):
    """Test with modified settings"""
    settings.DEBUG = True
    assert settings.DEBUG is True
```

### Common Failure Modes and Their Signatures

#### Symptom: "no such table" or "relation does not exist"

**Pytest output pattern**:
```
E   django.db.utils.OperationalError: no such table: blog_post
```

**Diagnostic clues**:
- Error occurs during database query
- Missing `@pytest.mark.django_db` decorator
- Test tries to access database without permission

**Root cause**: Test doesn't have database access enabled

**Solution**: Add `@pytest.mark.django_db` decorator to the test function

#### Symptom: Tests pass individually but fail together with "duplicate key" errors

**Pytest output pattern**:
```
E   django.db.utils.IntegrityError: UNIQUE constraint failed: auth_user.username
```

**Diagnostic clues**:
- Error mentions unique constraint
- Tests pass when run with `-k` to isolate them
- Involves creating users or other unique objects

**Root cause**: Fixtures creating objects with hardcoded unique values

**Solution**: Use dynamic values or ensure fixtures are function-scoped

#### Symptom: "Working outside of request context"

**Pytest output pattern**:
```
E   RuntimeError: Working outside of request context
```

**Diagnostic clues**:
- Error occurs when accessing request-dependent features
- Happens in middleware or view helpers
- Code works fine in actual application

**Root cause**: Test doesn't provide a request context

**Solution**: Use `RequestFactory` or test client to create proper request context (covered in section 14.5)

### When to Apply This Solution

**What it optimizes for**:
- Clean, readable test code
- Fast test execution with database rollback
- Fixture composition and reusability
- Better failure messages

**What it sacrifices**:
- Requires learning pytest-django specifics
- Different from Django's built-in testing approach
- May confuse developers familiar only with Django TestCase

**When to choose this approach**:
- New Django projects
- Projects already using pytest for other components
- When you need parametrized tests
- When fixture composition is important

**When to avoid this approach**:
- Team is strongly committed to Django's built-in testing
- Legacy codebase with extensive Django TestCase tests
- When consistency with Django documentation is critical

**Code characteristics**:
- **Setup complexity**: Medium (requires pytest-django configuration)
- **Maintenance burden**: Low (fixtures are reusable and composable)
- **Testability**: High (easy to test complex scenarios)

## Using Test Clients

## Using Test Clients

Test clients are the bridge between your test code and your web application. They simulate HTTP requests without requiring a running server. Both Flask and Django provide test clients, but they work differently. Let's explore how to use them effectively.

### The Test Client Abstraction

A test client provides methods that correspond to HTTP verbs:
- `client.get()` → GET request
- `client.post()` → POST request
- `client.put()` → PUT request
- `client.delete()` → DELETE request
- `client.patch()` → PATCH request

Each method returns a response object containing:
- `status_code` → HTTP status code
- `data` or `content` → Response body
- `headers` → Response headers
- `json()` or `get_json()` → Parsed JSON response

### Iteration 1: Basic Request Patterns

Let's explore common request patterns using our Flask task API:

```python
# test_client_basics.py
import pytest
from app import create_app

@pytest.fixture
def client():
    app = create_app({'TESTING': True})
    return app.test_client()

def test_get_request(client):
    """Test a simple GET request"""
    response = client.get('/tasks')
    
    assert response.status_code == 200
    assert isinstance(response.get_json(), list)

def test_post_request_with_json(client):
    """Test POST with JSON data"""
    response = client.post(
        '/tasks',
        json={'title': 'New task', 'description': 'Details'}
    )
    
    assert response.status_code == 201
    data = response.get_json()
    assert data['title'] == 'New task'
    assert data['description'] == 'Details'

def test_post_request_with_form_data(client):
    """Test POST with form data"""
    response = client.post(
        '/tasks',
        data={'title': 'New task'},
        content_type='application/x-www-form-urlencoded'
    )
    
    # This will fail because our API expects JSON
    assert response.status_code == 400
```

Run these tests:

```bash
$ pytest test_client_basics.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 3 items

test_client_basics.py::test_get_request PASSED                           [ 33%]
test_client_basics.py::test_post_request_with_json PASSED                [ 66%]
test_client_basics.py::test_post_request_with_form_data PASSED           [100%]

============================== 3 passed in 0.05s ===============================
```

These basic patterns work, but let's test a more complex scenario—following redirects:

```python
# Add to app.py
@app.route('/tasks/<int:task_id>/complete', methods=['POST'])
def complete_task(task_id):
    """Mark a task as complete and redirect to task detail"""
    task = app.tasks.get(task_id)
    
    if not task:
        return jsonify({'error': 'Task not found'}), 404
    
    task['completed'] = True
    
    # Redirect to the task detail page
    from flask import redirect, url_for
    return redirect(url_for('get_task', task_id=task_id), code=303)
```

```python
# test_client_basics.py (continued)
def test_redirect_not_followed_by_default(client):
    """Test that redirects are not followed by default"""
    # Create a task first
    create_response = client.post('/tasks', json={'title': 'Task'})
    task_id = create_response.get_json()['id']
    
    # Complete the task (triggers redirect)
    response = client.post(f'/tasks/{task_id}/complete')
    
    # By default, we get the redirect response
    assert response.status_code == 303
    assert 'Location' in response.headers
```

Run this test:

```bash
$ pytest test_client_basics.py::test_redirect_not_followed_by_default -v
```

**Output**:
```
============================= test session starts ==============================
collected 1 item

test_client_basics.py::test_redirect_not_followed_by_default PASSED     [100%]

============================== 1 passed in 0.02s ===============================
```

**Limitation preview**: We verified the redirect happened, but we didn't verify where it redirected to or what the final response contains. Let's test following the redirect:

```python
# test_client_basics.py (continued)
def test_redirect_followed(client):
    """Test following redirects"""
    # Create a task
    create_response = client.post('/tasks', json={'title': 'Task'})
    task_id = create_response.get_json()['id']
    
    # Complete the task and follow redirect
    response = client.post(
        f'/tasks/{task_id}/complete',
        follow_redirects=True
    )
    
    # Now we get the final response
    assert response.status_code == 200
    data = response.get_json()
    assert data['completed'] is True
```

### Iteration 2: Headers and Cookies

Web applications often use headers for authentication and cookies for session management. Let's test these:

```python
# Add to app.py
@app.route('/tasks/my', methods=['GET'])
def my_tasks():
    """Get tasks for the authenticated user"""
    auth_header = request.headers.get('Authorization')
    
    if not auth_header or not auth_header.startswith('Bearer '):
        return jsonify({'error': 'Authentication required'}), 401
    
    token = auth_header.split(' ')[1]
    
    # Simple token validation (in reality, you'd verify the token)
    if token != 'valid-token':
        return jsonify({'error': 'Invalid token'}), 401
    
    # Return all tasks (in reality, filter by user)
    return jsonify(list(app.tasks.values())), 200
```

```python
# test_client_headers.py
import pytest
from app import create_app

@pytest.fixture
def client():
    app = create_app({'TESTING': True})
    return app.test_client()

def test_request_without_auth_header(client):
    """Test that requests without auth header are rejected"""
    response = client.get('/tasks/my')
    
    assert response.status_code == 401
    assert 'error' in response.get_json()

def test_request_with_invalid_token(client):
    """Test that invalid tokens are rejected"""
    response = client.get(
        '/tasks/my',
        headers={'Authorization': 'Bearer invalid-token'}
    )
    
    assert response.status_code == 401

def test_request_with_valid_token(client):
    """Test authenticated request"""
    # Create a task first
    client.post('/tasks', json={'title': 'My task'})
    
    # Request with valid token
    response = client.get(
        '/tasks/my',
        headers={'Authorization': 'Bearer valid-token'}
    )
    
    assert response.status_code == 200
    tasks = response.get_json()
    assert len(tasks) == 1
    assert tasks[0]['title'] == 'My task'
```

Run these tests:

```bash
$ pytest test_client_headers.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 3 items

test_client_headers.py::test_request_without_auth_header PASSED          [ 33%]
test_client_headers.py::test_request_with_invalid_token PASSED           [ 66%]
test_client_headers.py::test_request_with_valid_token PASSED             [100%]

============================== 3 passed in 0.04s ===============================
```

Now let's test cookie handling:

```python
# Add to app.py
@app.route('/login', methods=['POST'])
def login():
    """Simple login that sets a session cookie"""
    from flask import make_response
    
    data = request.get_json()
    username = data.get('username')
    password = data.get('password')
    
    # Simple validation (in reality, check against database)
    if username == 'testuser' and password == 'testpass':
        response = make_response(jsonify({'message': 'Logged in'}), 200)
        response.set_cookie('session_id', 'abc123', httponly=True)
        return response
    
    return jsonify({'error': 'Invalid credentials'}), 401

@app.route('/profile', methods=['GET'])
def profile():
    """Get user profile (requires session cookie)"""
    session_id = request.cookies.get('session_id')
    
    if not session_id:
        return jsonify({'error': 'Not authenticated'}), 401
    
    # In reality, validate session_id against database
    if session_id != 'abc123':
        return jsonify({'error': 'Invalid session'}), 401
    
    return jsonify({'username': 'testuser', 'email': 'test@example.com'}), 200
```

```python
# test_client_cookies.py
import pytest
from app import create_app

@pytest.fixture
def client():
    app = create_app({'TESTING': True})
    return app.test_client()

def test_login_sets_cookie(client):
    """Test that login sets a session cookie"""
    response = client.post(
        '/login',
        json={'username': 'testuser', 'password': 'testpass'}
    )
    
    assert response.status_code == 200
    
    # Check that cookie was set
    assert 'session_id' in [cookie.name for cookie in client.cookie_jar]

def test_profile_requires_cookie(client):
    """Test that profile endpoint requires session cookie"""
    response = client.get('/profile')
    
    assert response.status_code == 401

def test_profile_with_cookie(client):
    """Test accessing profile with valid session"""
    # Login first to get cookie
    client.post(
        '/login',
        json={'username': 'testuser', 'password': 'testpass'}
    )
    
    # Cookie is automatically included in subsequent requests
    response = client.get('/profile')
    
    assert response.status_code == 200
    data = response.get_json()
    assert data['username'] == 'testuser'

def test_manual_cookie_setting(client):
    """Test manually setting cookies"""
    client.set_cookie('session_id', 'abc123')
    
    response = client.get('/profile')
    
    assert response.status_code == 200
```

Run these tests:

```bash
$ pytest test_client_cookies.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 4 items

test_client_cookies.py::test_login_sets_cookie PASSED                    [ 25%]
test_client_cookies.py::test_profile_requires_cookie PASSED              [ 50%]
test_client_cookies.py::test_profile_with_cookie PASSED                  [ 75%]
test_client_cookies.py::test_manual_cookie_setting PASSED                [100%]

============================== 4 passed in 0.05s ===============================
```

### Iteration 3: Testing File Uploads

Many web applications handle file uploads. Let's test this functionality:

```python
# Add to app.py
@app.route('/tasks/<int:task_id>/attachment', methods=['POST'])
def upload_attachment(task_id):
    """Upload a file attachment to a task"""
    task = app.tasks.get(task_id)
    
    if not task:
        return jsonify({'error': 'Task not found'}), 404
    
    if 'file' not in request.files:
        return jsonify({'error': 'No file provided'}), 400
    
    file = request.files['file']
    
    if file.filename == '':
        return jsonify({'error': 'Empty filename'}), 400
    
    # In reality, save the file to disk or cloud storage
    # For testing, just store metadata
    if 'attachments' not in task:
        task['attachments'] = []
    
    task['attachments'].append({
        'filename': file.filename,
        'content_type': file.content_type,
        'size': len(file.read())
    })
    
    return jsonify({'message': 'File uploaded', 'filename': file.filename}), 201
```

```python
# test_client_files.py
import pytest
from io import BytesIO
from app import create_app

@pytest.fixture
def client():
    app = create_app({'TESTING': True})
    return app.test_client()

def test_upload_file(client):
    """Test file upload"""
    # Create a task first
    create_response = client.post('/tasks', json={'title': 'Task with file'})
    task_id = create_response.get_json()['id']
    
    # Create a fake file
    file_content = b'This is test file content'
    file_data = {
        'file': (BytesIO(file_content), 'test.txt', 'text/plain')
    }
    
    # Upload the file
    response = client.post(
        f'/tasks/{task_id}/attachment',
        data=file_data,
        content_type='multipart/form-data'
    )
    
    assert response.status_code == 201
    data = response.get_json()
    assert data['filename'] == 'test.txt'
    
    # Verify the attachment was stored
    task_response = client.get(f'/tasks/{task_id}')
    task = task_response.get_json()
    assert len(task.get('attachments', [])) == 1
    assert task['attachments'][0]['filename'] == 'test.txt'
    assert task['attachments'][0]['size'] == len(file_content)

def test_upload_without_file(client):
    """Test upload endpoint without file"""
    create_response = client.post('/tasks', json={'title': 'Task'})
    task_id = create_response.get_json()['id']
    
    response = client.post(
        f'/tasks/{task_id}/attachment',
        data={},
        content_type='multipart/form-data'
    )
    
    assert response.status_code == 400
    assert 'error' in response.get_json()

def test_upload_empty_filename(client):
    """Test upload with empty filename"""
    create_response = client.post('/tasks', json={'title': 'Task'})
    task_id = create_response.get_json()['id']
    
    file_data = {
        'file': (BytesIO(b'content'), '', 'text/plain')
    }
    
    response = client.post(
        f'/tasks/{task_id}/attachment',
        data=file_data,
        content_type='multipart/form-data'
    )
    
    assert response.status_code == 400
```

Run these tests:

```bash
$ pytest test_client_files.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 3 items

test_client_files.py::test_upload_file PASSED                            [ 33%]
test_client_files.py::test_upload_without_file PASSED                    [ 66%]
test_client_files.py::test_upload_empty_filename PASSED                  [100%]

============================== 3 passed in 0.06s ===============================
```

### Django Test Client Differences

Django's test client has some differences from Flask's. Let's highlight the key ones:

```python
# test_django_client.py
import pytest
import json
from django.test import Client

@pytest.mark.django_db
def test_django_client_json_shortcut():
    """Django client has built-in JSON support"""
    client = Client()
    
    # Django 3.1+ supports direct JSON posting
    response = client.post(
        '/blog/posts/',
        {'title': 'Test'},
        content_type='application/json'
    )
    
    # Or use json.dumps explicitly
    response = client.post(
        '/blog/posts/',
        data=json.dumps({'title': 'Test'}),
        content_type='application/json'
    )

@pytest.mark.django_db
def test_django_client_follow_redirects():
    """Django uses 'follow' parameter instead of 'follow_redirects'"""
    client = Client()
    
    # Note: 'follow' not 'follow_redirects'
    response = client.post(
        '/some-endpoint/',
        follow=True
    )

@pytest.mark.django_db
def test_django_client_session_access():
    """Django client provides direct session access"""
    client = Client()
    
    # Access session directly
    session = client.session
    session['user_id'] = 123
    session.save()
    
    # Session persists across requests
    response = client.get('/profile/')

@pytest.mark.django_db
def test_django_client_force_login(django_user_model):
    """Django client has built-in authentication"""
    client = Client()
    user = django_user_model.objects.create_user(
        username='testuser',
        password='testpass'
    )
    
    # Force login without credentials
    client.force_login(user)
    
    response = client.get('/profile/')
    assert response.status_code == 200
```

### Common Failure Modes and Their Signatures

#### Symptom: "Content-Type header is not application/json"

**Pytest output pattern**:
```
E   AssertionError: assert 400 == 201
E   Response data: {'error': 'Content-Type must be application/json'}
```

**Diagnostic clues**:
- 400 Bad Request status
- Error message mentions Content-Type
- Forgot to specify `content_type='application/json'`

**Root cause**: Test client defaults to form-encoded data, but API expects JSON

**Solution**: Always specify `content_type='application/json'` when posting JSON data

#### Symptom: Cookies not persisting between requests

**Pytest output pattern**:
```
E   AssertionError: assert 401 == 200
E   Response: {'error': 'Not authenticated'}
```

**Diagnostic clues**:
- First request (login) succeeds
- Second request (authenticated endpoint) fails with 401
- Cookie was set but not sent in subsequent request

**Root cause**: Creating a new client instance between requests

**Solution**: Reuse the same client instance across related requests

#### Symptom: File upload fails with "No file provided"

**Pytest output pattern**:
```
E   AssertionError: assert 400 == 201
E   Response: {'error': 'No file provided'}
```

**Diagnostic clues**:
- Using `json=` parameter instead of `data=`
- Missing `content_type='multipart/form-data'`
- File not wrapped in tuple format

**Root cause**: Incorrect file upload format

**Solution**: Use `data={'file': (BytesIO(...), 'filename', 'content-type')}` with `content_type='multipart/form-data'`

### When to Apply This Solution

**What it optimizes for**:
- Fast test execution (no network overhead)
- Complete control over request parameters
- Easy testing of edge cases and error conditions
- Deterministic test behavior

**What it sacrifices**:
- Doesn't test actual HTTP server behavior
- Doesn't catch issues with reverse proxies or load balancers
- Doesn't test browser-specific behavior (JavaScript, rendering)

**When to choose this approach**:
- Unit and integration testing of API endpoints
- Testing request validation and error handling
- CI/CD pipelines where speed matters
- Testing authentication and authorization logic

**When to avoid this approach**:
- End-to-end testing of user workflows
- Testing JavaScript-heavy applications
- Performance testing under load
- Testing deployment configuration

**Code characteristics**:
- **Setup complexity**: Low (built into frameworks)
- **Maintenance burden**: Low (tests are simple and fast)
- **Testability**: High (easy to test all code paths)

## Fixtures for Web App Testing

## Fixtures for Web App Testing

Web application testing requires careful setup: creating test databases, initializing application instances, managing user sessions, and preparing test data. Fixtures are the key to making this setup reusable, composable, and maintainable.

### The Fixture Hierarchy for Web Testing

Web application fixtures typically follow this dependency hierarchy:

```
app (application instance)
  ↓
db (database connection)
  ↓
user (test user)
  ↓
authenticated_client (logged-in test client)
  ↓
test_data (domain objects: posts, tasks, etc.)
```

Let's build this hierarchy step by step, starting with our Flask task API.

### Iteration 1: Basic Application Fixtures

We've already seen simple app and client fixtures. Let's make them more sophisticated:

```python
# conftest.py
import pytest
from app import create_app

@pytest.fixture(scope='session')
def app():
    """Create application instance for the entire test session"""
    app = create_app({
        'TESTING': True,
        'SECRET_KEY': 'test-secret-key',
    })
    
    yield app

@pytest.fixture
def client(app):
    """Create a test client for each test"""
    return app.test_client()

@pytest.fixture
def runner(app):
    """Create a CLI test runner"""
    return app.test_cli_runner()
```

This works, but there's a problem. Let's test it:

```python
# test_fixtures_basic.py
def test_first_task(client):
    """Create a task"""
    response = client.post('/tasks', json={'title': 'First task'})
    assert response.get_json()['id'] == 1

def test_second_task(client):
    """Create another task"""
    response = client.post('/tasks', json={'title': 'Second task'})
    assert response.get_json()['id'] == 1  # Expect ID 1 in fresh state
```

Run these tests:

```bash
$ pytest test_fixtures_basic.py -v
```

**The complete output**:
```
============================= test session starts ==============================
collected 2 items

test_fixtures_basic.py::test_first_task PASSED                           [ 50%]
test_fixtures_basic.py::test_second_task FAILED                          [100%]

=================================== FAILURES ===================================
____________________________ test_second_task __________________________________

client = <FlaskClient <Flask 'app'>>

    def test_second_task(client):
        """Create another task"""
        response = client.post('/tasks', json={'title': 'Second task'})
>       assert response.get_json()['id'] == 1
E       AssertionError: assert 2 == 1
E        +  where 2 = {'id': 2, 'title': 'Second task', ...}['id']

test_fixtures_basic.py:8: AssertionError
=========================== 1 failed, 1 passed in 0.04s =======================
```

### Diagnostic Analysis: Session-Scoped State Pollution

**Let's parse this section by section**:

**1. The assertion introspection**:
```
E       AssertionError: assert 2 == 1
E        +  where 2 = {'id': 2, 'title': 'Second task', ...}['id']
```
   - What this tells us: The second test got ID 2, not ID 1
   - Key insight: State from the first test persisted

**2. Looking at our fixture**:
```python
@pytest.fixture(scope='session')
def app():
```
   - What this tells us: The app fixture is session-scoped
   - Key insight: The same app instance is reused across all tests

**Root cause identified**: Session-scoped fixtures are efficient but cause state pollution. The app's in-memory storage persists across tests.

**Why the current approach can't solve this**: We need the app to be session-scoped for performance (creating apps is expensive), but we need fresh state for each test.

**What we need**: A way to reset application state between tests while keeping the app instance itself session-scoped.

### Iteration 2: State Management with Context Managers

The solution is to separate the app instance (session-scoped) from the app state (function-scoped):

```python
# conftest.py (refactored)
import pytest
from app import create_app

@pytest.fixture(scope='session')
def app():
    """Create application instance once per session"""
    app = create_app({
        'TESTING': True,
        'SECRET_KEY': 'test-secret-key',
    })
    
    yield app

@pytest.fixture
def app_context(app):
    """Create a fresh application context for each test"""
    with app.app_context():
        # Reset application state
        app.tasks = {}
        app.task_id_counter = 1
        yield app

@pytest.fixture
def client(app_context):
    """Create a test client with fresh state"""
    return app_context.test_client()
```

Run the tests again:

```bash
$ pytest test_fixtures_basic.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 2 items

test_fixtures_basic.py::test_first_task PASSED                           [ 50%]
test_fixtures_basic.py::test_second_task PASSED                          [100%]

============================== 2 passed in 0.03s ===============================
```

**Expected vs. Actual improvement**:
- Tests now pass consistently
- App instance is created once (fast)
- State is reset for each test (isolated)

### Iteration 3: Database Fixtures

Real web applications use databases. Let's add database fixtures to our hierarchy:

```python
# conftest.py (with database support)
import pytest
from app import create_app, db as _db
from app.models import User, Task

@pytest.fixture(scope='session')
def app():
    """Create application instance with test database"""
    app = create_app({
        'TESTING': True,
        'SQLALCHEMY_DATABASE_URI': 'sqlite:///:memory:',
        'SQLALCHEMY_TRACK_MODIFICATIONS': False,
    })
    
    yield app

@pytest.fixture(scope='session')
def db(app):
    """Create database schema once per session"""
    with app.app_context():
        _db.create_all()
        yield _db
        _db.drop_all()

@pytest.fixture
def session(db):
    """Create a fresh database session for each test"""
    connection = db.engine.connect()
    transaction = connection.begin()
    
    # Create a session bound to the connection
    session = db.create_scoped_session(
        options={'bind': connection, 'binds': {}}
    )
    db.session = session
    
    yield session
    
    # Rollback transaction to reset database state
    transaction.rollback()
    connection.close()
    session.remove()

@pytest.fixture
def client(app, session):
    """Create a test client with database access"""
    return app.test_client()
```

Now we can write tests that use the database:

```python
# test_fixtures_database.py
import pytest
from app.models import User, Task

def test_create_user(session):
    """Test creating a user in the database"""
    user = User(username='testuser', email='test@example.com')
    session.add(user)
    session.commit()
    
    assert user.id is not None
    assert User.query.count() == 1

def test_database_is_empty_initially(session):
    """Verify database starts empty for each test"""
    assert User.query.count() == 0
    assert Task.query.count() == 0

def test_create_task_for_user(session):
    """Test creating a task associated with a user"""
    user = User(username='testuser', email='test@example.com')
    session.add(user)
    session.commit()
    
    task = Task(title='Test task', user_id=user.id)
    session.add(task)
    session.commit()
    
    assert task.id is not None
    assert task.user.username == 'testuser'
```

Run these tests:

```bash
$ pytest test_fixtures_database.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 3 items

test_fixtures_database.py::test_create_user PASSED                       [ 33%]
test_fixtures_database.py::test_database_is_empty_initially PASSED       [ 66%]
test_fixtures_database.py::test_create_task_for_user PASSED              [100%]

============================== 3 passed in 0.12s ===============================
```

### Iteration 4: User and Authentication Fixtures

Most web applications require authentication. Let's create fixtures for users and authenticated clients:

```python
# conftest.py (with authentication fixtures)
import pytest
from app.models import User

@pytest.fixture
def user(session):
    """Create a test user"""
    user = User(
        username='testuser',
        email='test@example.com'
    )
    user.set_password('testpass123')
    session.add(user)
    session.commit()
    return user

@pytest.fixture
def admin_user(session):
    """Create an admin user"""
    user = User(
        username='admin',
        email='admin@example.com',
        is_admin=True
    )
    user.set_password('adminpass123')
    session.add(user)
    session.commit()
    return user

@pytest.fixture
def authenticated_client(client, user):
    """Create a client authenticated as the test user"""
    # Login
    client.post('/login', json={
        'username': 'testuser',
        'password': 'testpass123'
    })
    return client

@pytest.fixture
def admin_client(client, admin_user):
    """Create a client authenticated as admin"""
    client.post('/login', json={
        'username': 'admin',
        'password': 'adminpass123'
    })
    return client
```

Now we can write tests that use authentication:

```python
# test_fixtures_auth.py
import pytest

def test_unauthenticated_access_denied(client):
    """Test that unauthenticated requests are rejected"""
    response = client.get('/tasks/my')
    assert response.status_code == 401

def test_authenticated_access_allowed(authenticated_client):
    """Test that authenticated requests succeed"""
    response = authenticated_client.get('/tasks/my')
    assert response.status_code == 200

def test_admin_access(admin_client):
    """Test admin-only endpoint"""
    response = admin_client.get('/admin/users')
    assert response.status_code == 200

def test_regular_user_cannot_access_admin(authenticated_client):
    """Test that regular users cannot access admin endpoints"""
    response = authenticated_client.get('/admin/users')
    assert response.status_code == 403
```

### Iteration 5: Test Data Fixtures

Complex tests often require multiple related objects. Let's create fixtures for common test data scenarios:

```python
# conftest.py (with test data fixtures)
import pytest
from app.models import Task, Comment

@pytest.fixture
def task(session, user):
    """Create a single test task"""
    task = Task(
        title='Test Task',
        description='Test description',
        user_id=user.id
    )
    session.add(task)
    session.commit()
    return task

@pytest.fixture
def tasks(session, user):
    """Create multiple test tasks"""
    task_list = []
    for i in range(5):
        task = Task(
            title=f'Task {i+1}',
            description=f'Description {i+1}',
            user_id=user.id,
            completed=(i % 2 == 0)  # Alternate completed status
        )
        session.add(task)
        task_list.append(task)
    session.commit()
    return task_list

@pytest.fixture
def task_with_comments(session, user, task):
    """Create a task with multiple comments"""
    for i in range(3):
        comment = Comment(
            content=f'Comment {i+1}',
            task_id=task.id,
            user_id=user.id
        )
        session.add(comment)
    session.commit()
    return task

@pytest.fixture
def multi_user_tasks(session, user, admin_user):
    """Create tasks for multiple users"""
    tasks = []
    
    # User's tasks
    for i in range(3):
        task = Task(
            title=f'User Task {i+1}',
            user_id=user.id
        )
        session.add(task)
        tasks.append(task)
    
    # Admin's tasks
    for i in range(2):
        task = Task(
            title=f'Admin Task {i+1}',
            user_id=admin_user.id
        )
        session.add(task)
        tasks.append(task)
    
    session.commit()
    return tasks
```

Now we can write tests that use complex test data:

```python
# test_fixtures_data.py
import pytest

def test_get_task(authenticated_client, task):
    """Test retrieving a single task"""
    response = authenticated_client.get(f'/tasks/{task.id}')
    
    assert response.status_code == 200
    data = response.get_json()
    assert data['title'] == 'Test Task'

def test_list_tasks(authenticated_client, tasks):
    """Test listing multiple tasks"""
    response = authenticated_client.get('/tasks')
    
    assert response.status_code == 200
    data = response.get_json()
    assert len(data['tasks']) == 5

def test_filter_completed_tasks(authenticated_client, tasks):
    """Test filtering by completion status"""
    response = authenticated_client.get('/tasks?completed=true')
    
    assert response.status_code == 200
    data = response.get_json()
    # 3 out of 5 tasks are completed (indices 0, 2, 4)
    assert len(data['tasks']) == 3

def test_task_with_comments(authenticated_client, task_with_comments):
    """Test retrieving a task with its comments"""
    response = authenticated_client.get(f'/tasks/{task_with_comments.id}/comments')
    
    assert response.status_code == 200
    data = response.get_json()
    assert len(data['comments']) == 3

def test_user_sees_only_own_tasks(authenticated_client, multi_user_tasks):
    """Test that users only see their own tasks"""
    response = authenticated_client.get('/tasks/my')
    
    assert response.status_code == 200
    data = response.get_json()
    # Should only see the 3 user tasks, not the 2 admin tasks
    assert len(data['tasks']) == 3
    assert all('User Task' in task['title'] for task in data['tasks'])
```

Run these tests:

```bash
$ pytest test_fixtures_data.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 5 items

test_fixtures_data.py::test_get_task PASSED                              [ 20%]
test_fixtures_data.py::test_list_tasks PASSED                            [ 40%]
test_fixtures_data.py::test_filter_completed_tasks PASSED                [ 60%]
test_fixtures_data.py::test_task_with_comments PASSED                    [ 80%]
test_fixtures_data.py::test_user_sees_only_own_tasks PASSED              [100%]

============================== 5 passed in 0.23s ===============================
```

### Fixture Composition Patterns

Let's explore advanced fixture composition patterns:

```python
# conftest.py (advanced patterns)
import pytest

@pytest.fixture
def api_headers():
    """Common API headers"""
    return {
        'Content-Type': 'application/json',
        'Accept': 'application/json'
    }

@pytest.fixture
def auth_headers(user):
    """Headers with authentication token"""
    token = user.generate_auth_token()
    return {
        'Content-Type': 'application/json',
        'Authorization': f'Bearer {token}'
    }

@pytest.fixture
def client_with_headers(client, api_headers):
    """Client that automatically includes API headers"""
    class ClientWithHeaders:
        def __init__(self, client, headers):
            self.client = client
            self.headers = headers
        
        def get(self, *args, **kwargs):
            kwargs.setdefault('headers', {}).update(self.headers)
            return self.client.get(*args, **kwargs)
        
        def post(self, *args, **kwargs):
            kwargs.setdefault('headers', {}).update(self.headers)
            return self.client.post(*args, **kwargs)
        
        # Add other HTTP methods as needed
    
    return ClientWithHeaders(client, api_headers)

@pytest.fixture(params=['user', 'admin'])
def any_authenticated_client(request, client, user, admin_user):
    """Parametrized fixture that tests with both user types"""
    if request.param == 'user':
        client.post('/login', json={
            'username': 'testuser',
            'password': 'testpass123'
        })
    else:
        client.post('/login', json={
            'username': 'admin',
            'password': 'adminpass123'
        })
    return client
```

### Common Failure Modes and Their Signatures

#### Symptom: "Working outside of application context"

**Pytest output pattern**:
```
E   RuntimeError: Working outside of application context.
E   This typically means that you attempted to use functionality that needed
E   to interface with the current application object in some way.
```

**Diagnostic clues**:
- Error occurs when accessing Flask features (current_app, g, url_for)
- Happens in fixtures or helper functions
- Code works fine in actual application

**Root cause**: Flask requires an application context for certain operations

**Solution**: Wrap code in `with app.app_context():` or use `app_context` fixture

#### Symptom: Database changes persist across tests

**Pytest output pattern**:
```
E   AssertionError: assert 3 == 0
E   Expected empty database but found 3 users
```

**Diagnostic clues**:
- First test passes, subsequent tests fail
- Database queries return unexpected data
- Tests pass when run individually

**Root cause**: Database session not properly rolled back between tests

**Solution**: Use transactional fixtures that rollback after each test

#### Symptom: Fixtures not found or not executing

**Pytest output pattern**:
```
E   fixture 'authenticated_client' not found
```

**Diagnostic clues**:
- Fixture is defined but pytest can't find it
- Fixture is in wrong file or wrong scope
- Import issues with conftest.py

**Root cause**: Fixture not in conftest.py or not properly scoped

**Solution**: Move fixtures to conftest.py or ensure proper import paths

### When to Apply This Solution

**What it optimizes for**:
- Test isolation and reliability
- Reusable test setup code
- Fast test execution through proper scoping
- Clear test intent (fixtures document what's needed)

**What it sacrifices**:
- Initial setup complexity
- Learning curve for fixture composition
- Potential over-abstraction if not careful

**When to choose this approach**:
- Projects with complex test setup requirements
- When multiple tests need similar setup
- When test isolation is critical
- When test speed matters (proper scoping)

**When to avoid this approach**:
- Very simple applications with minimal setup
- One-off tests that don't benefit from reuse
- When team is unfamiliar with pytest fixtures

**Code characteristics**:
- **Setup complexity**: Medium to High (requires understanding fixture scopes)
- **Maintenance burden**: Low (once set up, very maintainable)
- **Testability**: Very High (fixtures make complex scenarios easy to test)

## Testing Request Handlers

## Testing Request Handlers

Request handlers are the core of web applications—they receive HTTP requests, process them, and return responses. Testing them thoroughly requires understanding how to simulate different request scenarios, validate responses, and handle edge cases.

### The Anatomy of a Request Handler

A typical request handler:
1. Validates input (query params, body, headers)
2. Authenticates/authorizes the request
3. Performs business logic
4. Interacts with the database
5. Returns a formatted response

Let's test each of these aspects systematically.

### Iteration 1: Testing Input Validation

Input validation is the first line of defense. Let's test it thoroughly:

```python
# app/views.py
from flask import request, jsonify
from app import app, db
from app.models import Task

@app.route('/tasks', methods=['POST'])
def create_task():
    """Create a new task with validation"""
    data = request.get_json()
    
    # Validation: title is required
    if not data or 'title' not in data:
        return jsonify({'error': 'Title is required'}), 400
    
    # Validation: title must not be empty
    if not data['title'].strip():
        return jsonify({'error': 'Title cannot be empty'}), 400
    
    # Validation: title length
    if len(data['title']) > 200:
        return jsonify({'error': 'Title too long (max 200 characters)'}), 400
    
    # Validation: priority must be valid
    priority = data.get('priority', 'medium')
    if priority not in ['low', 'medium', 'high']:
        return jsonify({'error': 'Invalid priority'}), 400
    
    # Validation: due_date format
    due_date = data.get('due_date')
    if due_date:
        try:
            from datetime import datetime
            datetime.fromisoformat(due_date)
        except ValueError:
            return jsonify({'error': 'Invalid date format (use ISO 8601)'}), 400
    
    # Create task
    task = Task(
        title=data['title'].strip(),
        description=data.get('description', ''),
        priority=priority,
        due_date=due_date
    )
    db.session.add(task)
    db.session.commit()
    
    return jsonify(task.to_dict()), 201
```

Now let's write comprehensive validation tests:

```python
# test_request_handlers.py
import pytest
from datetime import datetime, timedelta

def test_create_task_success(client):
    """Test successful task creation"""
    response = client.post('/tasks', json={
        'title': 'Valid task',
        'description': 'Task description',
        'priority': 'high',
        'due_date': (datetime.now() + timedelta(days=7)).isoformat()
    })
    
    assert response.status_code == 201
    data = response.get_json()
    assert data['title'] == 'Valid task'
    assert data['priority'] == 'high'

def test_create_task_missing_title(client):
    """Test that missing title is rejected"""
    response = client.post('/tasks', json={
        'description': 'No title'
    })
    
    assert response.status_code == 400
    assert 'Title is required' in response.get_json()['error']

def test_create_task_empty_title(client):
    """Test that empty title is rejected"""
    response = client.post('/tasks', json={
        'title': '   '  # Only whitespace
    })
    
    assert response.status_code == 400
    assert 'cannot be empty' in response.get_json()['error']

def test_create_task_title_too_long(client):
    """Test that overly long titles are rejected"""
    response = client.post('/tasks', json={
        'title': 'x' * 201  # 201 characters
    })
    
    assert response.status_code == 400
    assert 'too long' in response.get_json()['error']

def test_create_task_invalid_priority(client):
    """Test that invalid priority is rejected"""
    response = client.post('/tasks', json={
        'title': 'Task',
        'priority': 'urgent'  # Not in allowed values
    })
    
    assert response.status_code == 400
    assert 'Invalid priority' in response.get_json()['error']

def test_create_task_invalid_date_format(client):
    """Test that invalid date format is rejected"""
    response = client.post('/tasks', json={
        'title': 'Task',
        'due_date': '2024-13-45'  # Invalid date
    })
    
    assert response.status_code == 400
    assert 'Invalid date format' in response.get_json()['error']

def test_create_task_with_defaults(client):
    """Test that defaults are applied correctly"""
    response = client.post('/tasks', json={
        'title': 'Minimal task'
    })
    
    assert response.status_code == 201
    data = response.get_json()
    assert data['priority'] == 'medium'  # Default
    assert data['description'] == ''  # Default
    assert data['due_date'] is None  # Default
```

Run these tests:

```bash
$ pytest test_request_handlers.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 7 items

test_request_handlers.py::test_create_task_success PASSED                [ 14%]
test_request_handlers.py::test_create_task_missing_title PASSED          [ 28%]
test_request_handlers.py::test_create_task_empty_title PASSED            [ 42%]
test_request_handlers.py::test_create_task_title_too_long PASSED         [ 57%]
test_request_handlers.py::test_create_task_invalid_priority PASSED       [ 71%]
test_request_handlers.py::test_create_task_invalid_date_format PASSED    [ 85%]
test_request_handlers.py::test_create_task_with_defaults PASSED          [100%]

============================== 7 passed in 0.18s ===============================
```

### Iteration 2: Testing Query Parameters and Filtering

Many endpoints accept query parameters for filtering, sorting, and pagination. Let's test these:

```python
# app/views.py (add this endpoint)
@app.route('/tasks', methods=['GET'])
def list_tasks():
    """List tasks with filtering and pagination"""
    # Get query parameters
    page = request.args.get('page', 1, type=int)
    per_page = request.args.get('per_page', 10, type=int)
    priority = request.args.get('priority')
    completed = request.args.get('completed', type=lambda v: v.lower() == 'true')
    sort_by = request.args.get('sort_by', 'created_at')
    order = request.args.get('order', 'desc')
    
    # Validate parameters
    if per_page > 100:
        return jsonify({'error': 'per_page cannot exceed 100'}), 400
    
    if sort_by not in ['created_at', 'due_date', 'priority', 'title']:
        return jsonify({'error': 'Invalid sort_by field'}), 400
    
    if order not in ['asc', 'desc']:
        return jsonify({'error': 'Invalid order (use asc or desc)'}), 400
    
    # Build query
    query = Task.query
    
    if priority:
        query = query.filter_by(priority=priority)
    
    if completed is not None:
        query = query.filter_by(completed=completed)
    
    # Apply sorting
    sort_column = getattr(Task, sort_by)
    if order == 'desc':
        sort_column = sort_column.desc()
    query = query.order_by(sort_column)
    
    # Paginate
    pagination = query.paginate(page=page, per_page=per_page, error_out=False)
    
    return jsonify({
        'tasks': [task.to_dict() for task in pagination.items],
        'total': pagination.total,
        'page': page,
        'per_page': per_page,
        'pages': pagination.pages
    }), 200
```

Now test the query parameter handling:

```python
# test_request_handlers.py (continued)
def test_list_tasks_default_pagination(client, tasks):
    """Test default pagination"""
    response = client.get('/tasks')
    
    assert response.status_code == 200
    data = response.get_json()
    assert 'tasks' in data
    assert 'total' in data
    assert data['per_page'] == 10

def test_list_tasks_custom_pagination(client, tasks):
    """Test custom pagination parameters"""
    response = client.get('/tasks?page=2&per_page=2')
    
    assert response.status_code == 200
    data = response.get_json()
    assert data['page'] == 2
    assert data['per_page'] == 2
    assert len(data['tasks']) <= 2

def test_list_tasks_filter_by_priority(client, tasks):
    """Test filtering by priority"""
    response = client.get('/tasks?priority=high')
    
    assert response.status_code == 200
    data = response.get_json()
    assert all(task['priority'] == 'high' for task in data['tasks'])

def test_list_tasks_filter_by_completed(client, tasks):
    """Test filtering by completion status"""
    response = client.get('/tasks?completed=true')
    
    assert response.status_code == 200
    data = response.get_json()
    assert all(task['completed'] is True for task in data['tasks'])

def test_list_tasks_sort_by_title(client, tasks):
    """Test sorting by title"""
    response = client.get('/tasks?sort_by=title&order=asc')
    
    assert response.status_code == 200
    data = response.get_json()
    titles = [task['title'] for task in data['tasks']]
    assert titles == sorted(titles)

def test_list_tasks_invalid_per_page(client):
    """Test that excessive per_page is rejected"""
    response = client.get('/tasks?per_page=101')
    
    assert response.status_code == 400
    assert 'cannot exceed 100' in response.get_json()['error']

def test_list_tasks_invalid_sort_field(client):
    """Test that invalid sort field is rejected"""
    response = client.get('/tasks?sort_by=invalid_field')
    
    assert response.status_code == 400
    assert 'Invalid sort_by' in response.get_json()['error']

def test_list_tasks_combined_filters(client, tasks):
    """Test combining multiple filters"""
    response = client.get('/tasks?priority=high&completed=false&sort_by=due_date')
    
    assert response.status_code == 200
    data = response.get_json()
    for task in data['tasks']:
        assert task['priority'] == 'high'
        assert task['completed'] is False
```

### Iteration 3: Testing Request Context and Headers

Some handlers depend on request context (headers, cookies, remote address). Let's test these:

```python
# app/views.py (add this endpoint)
@app.route('/tasks/<int:task_id>', methods=['PUT'])
def update_task(task_id):
    """Update a task with optimistic locking"""
    task = Task.query.get_or_404(task_id)
    data = request.get_json()
    
    # Check If-Match header for optimistic locking
    if_match = request.headers.get('If-Match')
    if if_match and if_match != task.etag:
        return jsonify({
            'error': 'Task was modified by another request',
            'current_etag': task.etag
        }), 412  # Precondition Failed
    
    # Check rate limiting based on IP
    from app.rate_limiter import check_rate_limit
    if not check_rate_limit(request.remote_addr):
        return jsonify({'error': 'Rate limit exceeded'}), 429
    
    # Update task
    if 'title' in data:
        task.title = data['title']
    if 'description' in data:
        task.description = data['description']
    if 'completed' in data:
        task.completed = data['completed']
    
    task.update_etag()
    db.session.commit()
    
    response = jsonify(task.to_dict())
    response.headers['ETag'] = task.etag
    return response, 200
```

Test the header-dependent behavior:

```python
# test_request_handlers.py (continued)
def test_update_task_without_etag(client, task):
    """Test update without ETag header"""
    response = client.put(f'/tasks/{task.id}', json={
        'title': 'Updated title'
    })
    
    assert response.status_code == 200
    assert response.get_json()['title'] == 'Updated title'

def test_update_task_with_matching_etag(client, task):
    """Test update with correct ETag"""
    response = client.put(
        f'/tasks/{task.id}',
        json={'title': 'Updated title'},
        headers={'If-Match': task.etag}
    )
    
    assert response.status_code == 200

def test_update_task_with_stale_etag(client, task):
    """Test update with outdated ETag"""
    old_etag = task.etag
    
    # Simulate another update
    task.title = 'Changed by someone else'
    task.update_etag()
    db.session.commit()
    
    # Try to update with old ETag
    response = client.put(
        f'/tasks/{task.id}',
        json={'title': 'My update'},
        headers={'If-Match': old_etag}
    )
    
    assert response.status_code == 412
    assert 'modified by another request' in response.get_json()['error']

def test_update_task_rate_limiting(client, task, monkeypatch):
    """Test rate limiting"""
    # Mock the rate limiter to return False
    def mock_check_rate_limit(ip):
        return False
    
    monkeypatch.setattr('app.rate_limiter.check_rate_limit', mock_check_rate_limit)
    
    response = client.put(f'/tasks/{task.id}', json={
        'title': 'Updated'
    })
    
    assert response.status_code == 429
    assert 'Rate limit exceeded' in response.get_json()['error']

def test_response_includes_etag_header(client, task):
    """Test that response includes ETag header"""
    response = client.put(f'/tasks/{task.id}', json={
        'title': 'Updated'
    })
    
    assert response.status_code == 200
    assert 'ETag' in response.headers
    assert response.headers['ETag'] == response.get_json()['etag']
```

### Iteration 4: Testing Error Handling and Edge Cases

Robust handlers handle errors gracefully. Let's test error scenarios:

```python
# app/views.py (add error handling)
@app.route('/tasks/<int:task_id>', methods=['DELETE'])
def delete_task(task_id):
    """Delete a task with proper error handling"""
    try:
        task = Task.query.get_or_404(task_id)
        
        # Check if task has dependencies
        if task.comments.count() > 0:
            return jsonify({
                'error': 'Cannot delete task with comments',
                'comment_count': task.comments.count()
            }), 409  # Conflict
        
        db.session.delete(task)
        db.session.commit()
        
        return '', 204  # No Content
        
    except Exception as e:
        db.session.rollback()
        app.logger.error(f'Error deleting task {task_id}: {str(e)}')
        return jsonify({'error': 'Internal server error'}), 500
```

Test error handling:

```python
# test_request_handlers.py (continued)
def test_delete_task_success(client, task):
    """Test successful task deletion"""
    response = client.delete(f'/tasks/{task.id}')
    
    assert response.status_code == 204
    assert response.data == b''
    
    # Verify task is deleted
    assert Task.query.get(task.id) is None

def test_delete_nonexistent_task(client):
    """Test deleting non-existent task"""
    response = client.delete('/tasks/99999')
    
    assert response.status_code == 404

def test_delete_task_with_comments(client, task_with_comments):
    """Test that tasks with comments cannot be deleted"""
    response = client.delete(f'/tasks/{task_with_comments.id}')
    
    assert response.status_code == 409
    data = response.get_json()
    assert 'Cannot delete' in data['error']
    assert data['comment_count'] > 0
    
    # Verify task still exists
    assert Task.query.get(task_with_comments.id) is not None

def test_delete_task_database_error(client, task, monkeypatch):
    """Test handling of database errors"""
    def mock_commit():
        raise Exception('Database connection lost')
    
    monkeypatch.setattr('app.db.session.commit', mock_commit)
    
    response = client.delete(f'/tasks/{task.id}')
    
    assert response.status_code == 500
    assert 'Internal server error' in response.get_json()['error']
    
    # Verify task still exists (rollback worked)
    assert Task.query.get(task.id) is not None
```

Run all request handler tests:

```bash
$ pytest test_request_handlers.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 23 items

test_request_handlers.py::test_create_task_success PASSED                [  4%]
test_request_handlers.py::test_create_task_missing_title PASSED          [  8%]
test_request_handlers.py::test_create_task_empty_title PASSED            [ 13%]
test_request_handlers.py::test_create_task_title_too_long PASSED         [ 17%]
test_request_handlers.py::test_create_task_invalid_priority PASSED       [ 21%]
test_request_handlers.py::test_create_task_invalid_date_format PASSED    [ 26%]
test_request_handlers.py::test_create_task_with_defaults PASSED          [ 30%]
test_request_handlers.py::test_list_tasks_default_pagination PASSED      [ 34%]
test_request_handlers.py::test_list_tasks_custom_pagination PASSED       [ 39%]
test_request_handlers.py::test_list_tasks_filter_by_priority PASSED      [ 43%]
test_request_handlers.py::test_list_tasks_filter_by_completed PASSED     [ 47%]
test_request_handlers.py::test_list_tasks_sort_by_title PASSED           [ 52%]
test_request_handlers.py::test_list_tasks_invalid_per_page PASSED        [ 56%]
test_request_handlers.py::test_list_tasks_invalid_sort_field PASSED      [ 60%]
test_request_handlers.py::test_list_tasks_combined_filters PASSED        [ 65%]
test_request_handlers.py::test_update_task_without_etag PASSED           [ 69%]
test_request_handlers.py::test_update_task_with_matching_etag PASSED     [ 73%]
test_request_handlers.py::test_update_task_with_stale_etag PASSED        [ 78%]
test_request_handlers.py::test_update_task_rate_limiting PASSED          [ 82%]
test_request_handlers.py::test_response_includes_etag_header PASSED      [ 86%]
test_request_handlers.py::test_delete_task_success PASSED                [ 91%]
test_request_handlers.py::test_delete_nonexistent_task PASSED            [ 95%]
test_request_handlers.py::test_delete_task_with_comments PASSED          [100%]

============================== 23 passed in 0.45s ===============================
```

### Common Failure Modes and Their Signatures

#### Symptom: Tests pass but real requests fail with 400 Bad Request

**Pytest output pattern**:
```
test_create_task PASSED
# But in production:
# 400 Bad Request: "The browser (or proxy) sent a request that this server could not understand."
```

**Diagnostic clues**:
- Tests use `json=` parameter correctly
- Real requests send different Content-Type
- Missing `Content-Type: application/json` header in production

**Root cause**: Test client automatically sets correct headers, but real clients might not

**Solution**: Add explicit header validation in handler and test both with and without correct headers

#### Symptom: Query parameter tests fail with unexpected types

**Pytest output pattern**:
```
E   TypeError: '>' not supported between instances of 'str' and 'int'
```

**Diagnostic clues**:
- Error occurs when comparing query parameter values
- Query parameters are strings by default
- Missing type conversion in handler

**Root cause**: Forgot to specify `type=` parameter in `request.args.get()`

**Solution**: Always specify type conversion: `request.args.get('page', 1, type=int)`

#### Symptom: Tests pass but handler modifies database unexpectedly

**Pytest output pattern**:
```
test_get_task PASSED
# But database shows unexpected changes
```

**Diagnostic clues**:
- GET request handler modifies data
- Tests don't verify database state after request
- Side effects not caught by response assertions

**Root cause**: Handler has unintended side effects

**Solution**: Add assertions that verify database state remains unchanged for read-only operations

### When to Apply This Solution

**What it optimizes for**:
- Comprehensive validation testing
- Edge case coverage
- Error handling verification
- Request/response contract testing

**What it sacrifices**:
- Test verbosity (many small tests)
- Setup complexity for complex scenarios
- Time to write comprehensive test suites

**When to choose this approach**:
- Public APIs that need robust validation
- Handlers with complex business logic
- Critical endpoints that handle sensitive data
- When debugging production issues

**When to avoid this approach**:
- Simple CRUD operations with minimal logic
- Internal APIs with trusted clients
- Prototype or proof-of-concept code
- When time constraints are severe

**Code characteristics**:
- **Setup complexity**: Medium (requires good fixtures)
- **Maintenance burden**: Medium (many tests to maintain)
- **Testability**: Very High (catches most handler bugs)

## Testing Middleware and Authentication

## Testing Middleware and Authentication

Middleware and authentication systems are the gatekeepers of web applications. They intercept requests before they reach handlers, making them critical to test thoroughly. Let's build a complete testing strategy for these cross-cutting concerns.

### The Reference Implementation: JWT Authentication Middleware

We'll implement and test a JWT-based authentication system with role-based access control:

```python
# app/middleware.py
from functools import wraps
from flask import request, jsonify, g
import jwt
from datetime import datetime, timedelta

SECRET_KEY = 'your-secret-key'

def generate_token(user_id, role='user'):
    """Generate a JWT token"""
    payload = {
        'user_id': user_id,
        'role': role,
        'exp': datetime.utcnow() + timedelta(hours=24),
        'iat': datetime.utcnow()
    }
    return jwt.encode(payload, SECRET_KEY, algorithm='HS256')

def token_required(f):
    """Decorator to require valid JWT token"""
    @wraps(f)
    def decorated(*args, **kwargs):
        token = None
        
        # Get token from Authorization header
        auth_header = request.headers.get('Authorization')
        if auth_header and auth_header.startswith('Bearer '):
            token = auth_header.split(' ')[1]
        
        if not token:
            return jsonify({'error': 'Token is missing'}), 401
        
        try:
            # Decode token
            payload = jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
            g.user_id = payload['user_id']
            g.role = payload['role']
        except jwt.ExpiredSignatureError:
            return jsonify({'error': 'Token has expired'}), 401
        except jwt.InvalidTokenError:
            return jsonify({'error': 'Invalid token'}), 401
        
        return f(*args, **kwargs)
    
    return decorated

def role_required(required_role):
    """Decorator to require specific role"""
    def decorator(f):
        @wraps(f)
        @token_required
        def decorated(*args, **kwargs):
            if g.role != required_role and g.role != 'admin':
                return jsonify({'error': 'Insufficient permissions'}), 403
            return f(*args, **kwargs)
        return decorated
    return decorator
```

Now let's create protected endpoints:

```python
# app/views.py
from app.middleware import token_required, role_required

@app.route('/protected', methods=['GET'])
@token_required
def protected_endpoint():
    """Endpoint that requires authentication"""
    return jsonify({
        'message': 'Access granted',
        'user_id': g.user_id,
        'role': g.role
    }), 200

@app.route('/admin/users', methods=['GET'])
@role_required('admin')
def admin_endpoint():
    """Endpoint that requires admin role"""
    return jsonify({
        'message': 'Admin access granted',
        'users': []  # Would fetch from database
    }), 200

@app.route('/moderator/reports', methods=['GET'])
@role_required('moderator')
def moderator_endpoint():
    """Endpoint that requires moderator role"""
    return jsonify({
        'message': 'Moderator access granted',
        'reports': []
    }), 200
```

### Iteration 1: Testing Basic Authentication

Let's start by testing the authentication decorator:

```python
# test_middleware.py
import pytest
from app import create_app
from app.middleware import generate_token

@pytest.fixture
def app():
    return create_app({'TESTING': True})

@pytest.fixture
def client(app):
    return app.test_client()

def test_protected_endpoint_without_token(client):
    """Test that requests without token are rejected"""
    response = client.get('/protected')
    
    assert response.status_code == 401
    assert 'Token is missing' in response.get_json()['error']

def test_protected_endpoint_with_valid_token(client):
    """Test that valid token grants access"""
    token = generate_token(user_id=123, role='user')
    
    response = client.get(
        '/protected',
        headers={'Authorization': f'Bearer {token}'}
    )
    
    assert response.status_code == 200
    data = response.get_json()
    assert data['user_id'] == 123
    assert data['role'] == 'user'

def test_protected_endpoint_with_malformed_header(client):
    """Test that malformed Authorization header is rejected"""
    response = client.get(
        '/protected',
        headers={'Authorization': 'InvalidFormat token123'}
    )
    
    assert response.status_code == 401

def test_protected_endpoint_with_invalid_token(client):
    """Test that invalid token is rejected"""
    response = client.get(
        '/protected',
        headers={'Authorization': 'Bearer invalid.token.here'}
    )
    
    assert response.status_code == 401
    assert 'Invalid token' in response.get_json()['error']
```

Run these tests:

```bash
$ pytest test_middleware.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 4 items

test_middleware.py::test_protected_endpoint_without_token PASSED         [ 25%]
test_middleware.py::test_protected_endpoint_with_valid_token PASSED      [ 50%]
test_middleware.py::test_protected_endpoint_with_malformed_header PASSED [ 75%]
test_middleware.py::test_protected_endpoint_with_invalid_token PASSED    [100%]

============================== 4 passed in 0.08s ===============================
```

**Limitation preview**: We've tested basic authentication, but we haven't tested token expiration. Let's see what happens with expired tokens:

```python
# test_middleware.py (continued)
def test_protected_endpoint_with_expired_token(client):
    """Test that expired token is rejected"""
    # Generate a token that's already expired
    import jwt
    from datetime import datetime, timedelta
    
    payload = {
        'user_id': 123,
        'role': 'user',
        'exp': datetime.utcnow() - timedelta(hours=1),  # Expired 1 hour ago
        'iat': datetime.utcnow() - timedelta(hours=2)
    }
    expired_token = jwt.encode(payload, 'your-secret-key', algorithm='HS256')
    
    response = client.get(
        '/protected',
        headers={'Authorization': f'Bearer {expired_token}'}
    )
    
    assert response.status_code == 401
    assert 'expired' in response.get_json()['error'].lower()
```

Run this test:

```bash
$ pytest test_middleware.py::test_protected_endpoint_with_expired_token -v
```

**Output**:
```
============================= test session starts ==============================
collected 1 item

test_middleware.py::test_protected_endpoint_with_expired_token PASSED    [100%]

============================== 1 passed in 0.02s ===============================
```

### Iteration 2: Testing Role-Based Access Control

Now let's test the role-based authorization:

```python
# test_middleware.py (continued)
def test_admin_endpoint_with_admin_token(client):
    """Test that admin token grants access to admin endpoint"""
    token = generate_token(user_id=1, role='admin')
    
    response = client.get(
        '/admin/users',
        headers={'Authorization': f'Bearer {token}'}
    )
    
    assert response.status_code == 200
    assert 'Admin access granted' in response.get_json()['message']

def test_admin_endpoint_with_user_token(client):
    """Test that regular user cannot access admin endpoint"""
    token = generate_token(user_id=123, role='user')
    
    response = client.get(
        '/admin/users',
        headers={'Authorization': f'Bearer {token}'}
    )
    
    assert response.status_code == 403
    assert 'Insufficient permissions' in response.get_json()['error']

def test_moderator_endpoint_with_moderator_token(client):
    """Test that moderator token grants access to moderator endpoint"""
    token = generate_token(user_id=2, role='moderator')
    
    response = client.get(
        '/moderator/reports',
        headers={'Authorization': f'Bearer {token}'}
    )
    
    assert response.status_code == 200

def test_moderator_endpoint_with_admin_token(client):
    """Test that admin can access moderator endpoints"""
    token = generate_token(user_id=1, role='admin')
    
    response = client.get(
        '/moderator/reports',
        headers={'Authorization': f'Bearer {token}'}
    )
    
    assert response.status_code == 200

def test_moderator_endpoint_with_user_token(client):
    """Test that regular user cannot access moderator endpoint"""
    token = generate_token(user_id=123, role='user')
    
    response = client.get(
        '/moderator/reports',
        headers={'Authorization': f'Bearer {token}'}
    )
    
    assert response.status_code == 403
```

Run all middleware tests:

```bash
$ pytest test_middleware.py -v
```

**Output**:
```
============================= test session starts ==============================
collected 10 items

test_middleware.py::test_protected_endpoint_without_token PASSED         [ 10%]
test_middleware.py::test_protected_endpoint_with_valid_token PASSED      [ 20%]
test_middleware.py::test_protected_endpoint_with_malformed_header PASSED [ 30%]
test_middleware.py::test_protected_endpoint_with_invalid_token PASSED    [ 40%]
test_middleware.py::test_protected_endpoint_with_expired_token PASSED    [ 50%]
test_middleware.py::test_admin_endpoint_with_admin_token PASSED          [ 60%]
test_middleware.py::test_admin_endpoint_with_user_token PASSED           [ 70%]
test_middleware.py::test_moderator_endpoint_with_moderator_token PASSED  [ 80%]
test_middleware.py::test_moderator_endpoint_with_admin_token PASSED      [ 90%]
test_middleware.py::test_moderator_endpoint_with_user_token PASSED       [100%]

============================== 10 passed in 0.15s ===============================
```

### Iteration 3: Testing Request/Response Middleware

Let's test middleware that processes all requests and responses:

```python
# app/middleware.py (add logging middleware)
from flask import request, g
import time
import logging

logger = logging.getLogger(__name__)

@app.before_request
def before_request():
    """Log request details and start timer"""
    g.start_time = time.time()
    logger.info(f'Request: {request.method} {request.path}')
    
    # Add request ID for tracing
    g.request_id = request.headers.get('X-Request-ID', 'unknown')

@app.after_request
def after_request(response):
    """Log response details and add headers"""
    # Calculate request duration
    if hasattr(g, 'start_time'):
        duration = time.time() - g.start_time
        response.headers['X-Request-Duration'] = str(duration)
    
    # Add request ID to response
    if hasattr(g, 'request_id'):
        response.headers['X-Request-ID'] = g.request_id
    
    # Add security headers
    response.headers['X-Content-Type-Options'] = 'nosniff'
    response.headers['X-Frame-Options'] = 'DENY'
    
    logger.info(f'Response: {response.status_code}')
    return response

@app.errorhandler(404)
def not_found(error):
    """Custom 404 handler"""
    return jsonify({
        'error': 'Resource not found',
        'path': request.path
    }), 404

@app.errorhandler(500)
def internal_error(error):
    """Custom 500 handler"""
    logger.error(f'Internal error: {str(error)}')
    return jsonify({
        'error': 'Internal server error',
        'request_id': g.get('request_id', 'unknown')
    }), 500
```

Test the middleware behavior:

```python
# test_middleware_hooks.py
import pytest
from app import create_app

@pytest.fixture
def app():
    return create_app({'TESTING': True})

@pytest.fixture
def client(app):
    return app.test_client()

def test_request_id_header_added_to_response(client):
    """Test that request ID is added to response"""
    response = client.get(
        '/tasks',
        headers={'X-Request-ID': 'test-123'}
    )
    
    assert 'X-Request-ID' in response.headers
    assert response.headers['X-Request-ID'] == 'test-123'

def test_request_duration_header_added(client):
    """Test that request duration is tracked"""
    response = client.get('/tasks')
    
    assert 'X-Request-Duration' in response.headers
    duration = float(response.headers['X-Request-Duration'])
    assert duration > 0

def test_security_headers_added(client):
    """Test that security headers are added"""
    response = client.get('/tasks')
    
    assert response.headers['X-Content-Type-Options'] == 'nosniff'
    assert response.headers['X-Frame-Options'] == 'DENY'

def test_custom_404_handler(client):
    """Test custom 404 error handler"""
    response = client.get('/nonexistent-endpoint')
    
    assert response.status_code == 404
    data = response.get_json()
    assert 'Resource not found' in data['error']
    assert data['path'] == '/nonexistent-endpoint'

def test_custom_500_handler(client, monkeypatch):
    """Test custom 500 error handler"""
    # Create an endpoint that raises an exception
    @app.route('/error-test')
    def error_endpoint():
        raise Exception('Test error')
    
    response = client.get('/error-test')
    
    assert response.status_code == 500
    data = response.get_json()
    assert 'Internal server error' in data['error']
    assert 'request_id' in data

def test_middleware_execution_order(client, caplog):
    """Test that middleware executes in correct order"""
    import logging
    caplog.set_level(logging.INFO)
    
    response = client.get('/tasks')
    
    # Check log messages appear in correct order
    log_messages = [record.message for record in caplog.records]
    request_log = next((msg for msg in log_messages if 'Request:' in msg), None)
    response_log = next((msg for msg in log_messages if 'Response:' in msg), None)
    
    assert request_log is not None
    assert response_log is not None
    assert log_messages.index(request_log) < log_messages.index(response_log)
```

### Iteration 4: Testing Authentication Fixtures

Create reusable fixtures for authenticated testing:

```python
# conftest.py (add authentication fixtures)
import pytest
from app.middleware import generate_token

@pytest.fixture
def user_token():
    """Generate a token for a regular user"""
    return generate_token(user_id=123, role='user')

@pytest.fixture
def admin_token():
    """Generate a token for an admin user"""
    return generate_token(user_id=1, role='admin')

@pytest.fixture
def moderator_token():
    """Generate a token for a moderator"""
    return generate_token(user_id=2, role='moderator')

@pytest.fixture
def auth_headers(user_token):
    """Generate headers with user authentication"""
    return {'Authorization': f'Bearer {user_token}'}

@pytest.fixture
def admin_headers(admin_token):
    """Generate headers with admin authentication"""
    return {'Authorization': f'Bearer {admin_token}'}

@pytest.fixture
def authenticated_client(client, auth_headers):
    """Create a client with authentication headers pre-configured"""
    class AuthenticatedClient:
        def __init__(self, client, headers):
            self.client = client
            self.headers = headers
        
        def get(self, *args, **kwargs):
            kwargs.setdefault('headers', {}).update(self.headers)
            return self.client.get(*args, **kwargs)
        
        def post(self, *args, **kwargs):
            kwargs.setdefault('headers', {}).update(self.headers)
            return self.client.post(*args, **kwargs)
        
        def put(self, *args, **kwargs):
            kwargs.setdefault('headers', {}).update(self.headers)
            return self.client.put(*args, **kwargs)
        
        def delete(self, *args, **kwargs):
            kwargs.setdefault('headers', {}).update(self.headers)
            return self.client.delete(*args, **kwargs)
    
    return AuthenticatedClient(client, auth_headers)
```

Now tests become much cleaner:

```python
# test_with_auth_fixtures.py
import pytest

def test_create_task_with_auth_fixture(authenticated_client):
    """Test using authenticated client fixture"""
    response = authenticated_client.post('/tasks', json={
        'title': 'New task'
    })
    
    assert response.status_code == 201

def test_admin_endpoint_with_admin_headers(client, admin_headers):
    """Test using admin headers fixture"""
    response = client.get('/admin/users', headers=admin_headers)
    
    assert response.status_code == 200

def test_multiple_roles(client, user_token, admin_token):
    """Test comparing behavior across roles"""
    # User attempt
    user_response = client.get(
        '/admin/users',
        headers={'Authorization': f'Bearer {user_token}'}
    )
    assert user_response.status_code == 403
    
    # Admin attempt
    admin_response = client.get(
        '/admin/users',
        headers={'Authorization': f'Bearer {admin_token}'}
    )
    assert admin_response.status_code == 200
```

### Common Failure Modes and Their Signatures

#### Symptom: Middleware not executing for certain routes

**Pytest output pattern**:
```
E   AssertionError: assert 'X-Request-ID' not in response.headers
```

**Diagnostic clues**:
- Middleware works for some endpoints but not others
- Static files or error handlers bypass middleware
- Blueprint-specific middleware not registered correctly

**Root cause**: Middleware registered after routes or on wrong blueprint

**Solution**: Register middleware on app instance before registering blueprints

#### Symptom: Authentication works in tests but fails in production

**Pytest output pattern**:
```
test_protected_endpoint PASSED
# But in production: 401 Unauthorized
```

**Diagnostic clues**:
- Tests use hardcoded secret key
- Production uses environment variable
- Token generated with different key than validation

**Root cause**: Secret key mismatch between test and production

**Solution**: Use same configuration mechanism in tests as production

#### Symptom: Middleware modifies request but handler doesn't see changes

**Pytest output pattern**:
```
E   AssertionError: assert hasattr(g, 'user_id') is True
```

**Diagnostic clues**:
- Middleware sets `g.user_id`
- Handler can't access `g.user_id`
- Works in some tests but not others

**Root cause**: Application context not properly managed in tests

**Solution**: Ensure tests use `app.app_context()` or proper client fixtures

### The Journey: From Problem to Solution

| Iteration | Failure Mode                    | Technique Applied                | Result                      |
| --------- | ------------------------------- | -------------------------------- | --------------------------- |
| 0         | No authentication testing       | None                             | Insecure endpoints          |
| 1         | Basic auth not tested           | Token generation fixtures        | Basic security verified     |
| 2         | Role-based access not tested    | Role-specific fixtures           | Authorization verified      |
| 3         | Middleware hooks not tested     | Request/response inspection      | Cross-cutting concerns tested |
| 4         | Repetitive auth setup           | Authenticated client fixtures    | Clean, maintainable tests   |

### Final Implementation

Here's the complete, production-ready testing setup:

```python
# conftest.py (complete version)
import pytest
from app import create_app
from app.middleware import generate_token

@pytest.fixture(scope='session')
def app():
    """Create application for testing"""
    return create_app({
        'TESTING': True,
        'SECRET_KEY': 'test-secret-key',
        'SQLALCHEMY_DATABASE_URI': 'sqlite:///:memory:'
    })

@pytest.fixture
def client(app):
    """Create test client"""
    return app.test_client()

# Authentication fixtures
@pytest.fixture
def user_token():
    return generate_token(user_id=123, role='user')

@pytest.fixture
def admin_token():
    return generate_token(user_id=1, role='admin')

@pytest.fixture
def moderator_token():
    return generate_token(user_id=2, role='moderator')

@pytest.fixture
def auth_headers(user_token):
    return {'Authorization': f'Bearer {user_token}'}

@pytest.fixture
def admin_headers(admin_token):
    return {'Authorization': f'Bearer {admin_token}'}

@pytest.fixture
def authenticated_client(client, auth_headers):
    """Client with automatic authentication"""
    class AuthenticatedClient:
        def __init__(self, client, headers):
            self.client = client
            self.headers = headers
        
        def request(self, method, *args, **kwargs):
            kwargs.setdefault('headers', {}).update(self.headers)
            return getattr(self.client, method)(*args, **kwargs)
        
        def get(self, *args, **kwargs):
            return self.request('get', *args, **kwargs)
        
        def post(self, *args, **kwargs):
            return self.request('post', *args, **kwargs)
        
        def put(self, *args, **kwargs):
            return self.request('put', *args, **kwargs)
        
        def delete(self, *args, **kwargs):
            return self.request('delete', *args, **kwargs)
    
    return AuthenticatedClient(client, auth_headers)
```

### Decision Framework: Which Approach When?

| Scenario                          | Approach                        | Why                                    |
| --------------------------------- | ------------------------------- | -------------------------------------- |
| Testing public endpoints          | Plain `client` fixture          | No authentication needed               |
| Testing user-specific endpoints   | `authenticated_client` fixture  | Automatic auth headers                 |
| Testing multiple roles            | Individual token fixtures       | Explicit role comparison               |
| Testing auth failures             | Manual token generation         | Need invalid/expired tokens            |
| Testing middleware behavior       | Request/response inspection     | Verify headers and timing              |
| Testing error handlers            | Trigger errors explicitly       | Verify error response format           |

### Lessons Learned

**Key insights from testing middleware and authentication**:

1. **Fixtures are essential**: Authentication testing requires many fixtures, but they make tests clean and maintainable

2. **Test the negative cases**: Most bugs are in error handling—test invalid tokens, expired tokens, wrong roles

3. **Middleware order matters**: Test that middleware executes in the correct sequence

4. **Context is critical**: Flask's `g` object and application context must be properly managed in tests

5. **Security headers matter**: Don't forget to test that security headers are added correctly

6. **Composition over duplication**: Build higher-level fixtures from lower-level ones (authenticated_client from auth_headers from token)

### When to Apply This Solution

**What it optimizes for**:
- Security verification
- Authorization testing
- Cross-cutting concern validation
- Middleware behavior verification

**What it sacrifices**:
- Test complexity (many fixtures)
- Setup time (authentication infrastructure)
- Learning curve (understanding middleware execution)

**When to choose this approach**:
- Applications with authentication requirements
- APIs with role-based access control
- Systems with complex middleware chains
- Security-critical applications

**When to avoid this approach**:
- Public APIs with no authentication
- Simple applications without middleware
- Prototype or proof-of-concept code
- When security is not a concern

**Code characteristics**:
- **Setup complexity**: High (requires authentication infrastructure)
- **Maintenance burden**: Medium (fixtures need updates when auth changes)
- **Testability**: Very High (comprehensive security testing)
