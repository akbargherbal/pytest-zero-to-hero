# Chapter 3: Setting Up Your Testing Environment

## Project Structure Best Practices

## The Blueprint for a Testable Project

Before writing a single line of test code, the structure of your project can either set you up for success or for future headaches. A well-organized project is easy to navigate, easy to test, and easy for others to understand. An unorganized one becomes a tangled mess where application code and test code are indistinguishable.

Let's illuminate this by showing the pits. Imagine a project folder that looks like this:

```
messy_project/
├── my_app.py
├── test_my_app.py
├── utils.py
├── test_utils.py
├── data.json
└── README.md
```

While this might work for a tiny script, it quickly becomes unmanageable. Which files are part of the application? Which are tests? Where do new tests go? This flat structure creates confusion.

Now, let's look at a clean, standard, and scalable structure.

### The `src` Layout

The most robust and recommended structure for modern Python projects is the "src layout." It creates a clear separation between your actual application code and everything else (like tests, documentation, and configuration).

Here is the blueprint we will use for our examples:

```
wallet_project/
├── src/
│   └── wallet/
│       ├── __init__.py
│       ├── wallet.py
│       └── exceptions.py
├── tests/
│   ├── __init__.py
│   └── test_wallet.py
├── .gitignore
├── pyproject.toml
└── README.md
```

Let's break down why this is so effective:

1.  **`src/` directory**: This contains your *installable* Python package. Everything inside `src` is what you would distribute if you were publishing your project. This clear boundary prevents many common packaging and import path issues.
2.  **`wallet/`**: This is the actual Python package. The name you choose here is what users will `import` (e.g., `from wallet import Wallet`).
3.  **`tests/` directory**: This is the home for all your tests. It lives at the root of the project, parallel to `src`, making it clear that tests are not part of the installable application code but are a critical part of the development process.
4.  **Configuration Files**: Files like `pyproject.toml` (which we'll cover shortly) and `.gitignore` live at the root level.

This structure isn't just a suggestion; it's a convention that solves real-world problems. It ensures your tests are run against the *installed* version of your code, just as a user would experience it, preventing tests that pass locally but fail in production because of tricky import paths.

## Creating a tests/ Directory

## A Dedicated Home for Your Tests

As we saw in the best-practice layout, tests should live in their own top-level directory, almost always named `tests/`. Why not just put `test_wallet.py` next to `wallet.py`?

**The Principle of Separation:** Your tests are for developers, not for end-users. When you package your `wallet` application for distribution, you don't want to include the hundreds of test files. Placing them in a separate `tests/` directory makes it trivial to exclude them from the final package, keeping it lean.

Let's create this structure. Assume you are in your main project directory (`wallet_project/`).

First, create the source directory and the package within it. The `-p` flag creates parent directories as needed.

```bash
mkdir -p src/wallet
```

Next, create the `tests/` directory at the same level as `src/`.

```bash
mkdir tests
```

Now, let's create some placeholder files to make our project tangible. We'll use `touch` to create empty files.

```bash
# Create empty __init__.py files to mark them as Python packages
touch src/wallet/__init__.py
touch tests/__init__.py

# Create our main application and test files
touch src/wallet/wallet.py
touch tests/test_wallet.py
```

Your project structure should now match the blueprint from the previous section.

### How Pytest Finds the `tests` Directory

In Chapter 2, we discussed test discovery. By default, pytest looks for tests in the current directory and its subdirectories. When you run `pytest` from your project root (`wallet_project/`), it will automatically find and explore the `tests/` directory, discovering `test_wallet.py` and any other files that follow the `test_*.py` or `*_test.py` naming convention.

This convention is powerful because it requires zero configuration to get started. Pytest just *knows* where to look. Later in this chapter, we'll see how to explicitly tell pytest where your tests are, which is a best practice for larger projects.

## Using Virtual Environments

## Isolating Your Project's World

Imagine you are working on two projects. Project A needs version 1.0 of a library called `requests`, while Project B needs the brand-new version 2.0. If you install these libraries globally on your computer, you have a conflict. Upgrading for Project B will break Project A, and downgrading for Project A will break Project B. This is often called "dependency hell."

**A virtual environment is a self-contained directory that holds a specific version of Python and all the specific libraries your project needs.** It's a private workspace for each project, preventing any conflicts.

### The Wrong Way: Global Installation

You might be tempted to just run `pip install pytest` in your main system terminal. **Don't do this.** This installs pytest globally. While it might seem to work at first, it will inevitably lead to the dependency conflicts described above. It's a classic pitfall that seems easier initially but creates significant problems down the road.

### The Right Way: `venv`

Python comes with a built-in module called `venv` for creating virtual environments. Let's create one for our `wallet_project`.

Navigate to your project's root directory (`wallet_project/`).

```bash
# Create a virtual environment in a directory named .venv
# Using .venv is a common convention
python3 -m venv .venv
```

You will now see a new `.venv/` directory in your project. This folder contains a copy of the Python interpreter and is ready to hold your project's dependencies. You should add `.venv` to your `.gitignore` file to keep it out of version control.

### Activating the Virtual Environment

Creating the environment isn't enough; you need to "activate" it to tell your shell session to use it.

**On macOS and Linux:**

```bash
source .venv/bin/activate
```

**On Windows (Command Prompt):**

```bash
.venv\Scripts\activate.bat
```

**On Windows (PowerShell):**

```bash
.venv\Scripts\Activate.ps1
```

Once activated, you'll typically see the name of the environment in your shell prompt, like `(.venv) $`. This confirms that any Python or pip commands you run will now operate *inside* this isolated environment.

### Installing Dependencies

With your environment active, you can now safely install pytest and any other libraries.

```bash
# This pip is the one inside .venv/bin/
pip install pytest

# Let's say our wallet needs an external library
pip install requests
```

To prove the isolation, you can deactivate the environment.

```bash
deactivate
```

Now, if you try to run `pytest`, the command will likely fail (unless you made the mistake of installing it globally). This confirms your project's dependencies are neatly contained within `.venv`. Always remember to activate your virtual environment before working on your project.

## Pytest Configuration Files (pytest.ini, setup.cfg, pyproject.toml)

## Telling Pytest How to Behave

While pytest works brilliantly out of the box, you'll soon want to customize its behavior. You can pass options on the command line (like `pytest -v`), but typing these every time is tedious.

Pytest configuration files allow you to set default options and configure project-specific settings. Pytest looks for one of these files in your project root, in order of preference:

1.  `pyproject.toml`
2.  `pytest.ini`
3.  `tox.ini`
4.  `setup.cfg`

Let's explore the three most common choices, showing multiple paths to the same destination. We'll configure a simple option: always add `-v` (verbose) to every `pytest` run.

### `pytest.ini`

This is the simplest, most direct way to configure pytest. It's an INI-formatted file dedicated solely to pytest settings.

Create a file named `pytest.ini` in your project root.

```ini
# pytest.ini
[pytest]
addopts = -v
```

Now, when you run `pytest`, it will automatically behave as if you had typed `pytest -v`.

-   **Pros**: Simple, clear, and focused only on pytest.
-   **Cons**: Adds another configuration file to your project root.

### `setup.cfg`

This file is a holdover from the `setuptools` packaging system. If your project already has one for packaging metadata, you can add a `[tool:pytest]` section to it.

```ini
# setup.cfg
[tool:pytest]
addopts = -v
```

-   **Pros**: Consolidates configuration if you're already using `setup.cfg`.
-   **Cons**: `setuptools` is being superseded by newer standards. This is now considered a legacy approach.

### `pyproject.toml` (Recommended)

This is the modern, standardized way to configure Python tools. It's defined in PEP 518 and is designed to be the single configuration file for your entire project, from build systems to linters to test runners.

Create a file named `pyproject.toml` in your project root.

```toml
# pyproject.toml
[tool.pytest.ini_options]
addopts = "-v"
```

-   **Pros**: The official, future-proof standard. Consolidates all tool configuration into one file, reducing clutter.
-   **Cons**: The nested structure (`[tool.pytest.ini_options]`) is slightly more verbose.

**Our Recommendation:** For any new project, use `pyproject.toml`. It is the clear direction the Python ecosystem is heading, and most modern tools support it. We will use `pyproject.toml` for all examples going forward.

## Common Configuration Options

## Fine-Tuning Your Test Suite

Let's build on our `pyproject.toml` file and add some of the most useful configuration options. We'll introduce them one at a time, explaining the problem each one solves.

Our starting `pyproject.toml`:

```toml
# pyproject.toml
[tool.pytest.ini_options]
addopts = "-v"
```

### `testpaths`: Specifying Where to Find Tests

By default, pytest searches everywhere. In a large project, this can be slow. It might also accidentally pick up tests from a virtual environment directory or other places you don't intend. It's better to be explicit.

The `testpaths` option tells pytest exactly where to look.

```toml
# pyproject.toml
[tool.pytest.ini_options]
addopts = "-v"
testpaths = [
    "tests",
]
```

Now, pytest will *only* look for tests inside the `tests/` directory, making discovery faster and more predictable.

### `python_files`, `python_classes`, `python_functions`: Customizing Discovery

Pytest's default discovery (looking for `test_*` and `*_test`) is excellent, but sometimes you need to change it. For example, a project might have a convention of naming test files `check_*.py`.

You can override the default patterns:

```toml
# pyproject.toml
[tool.pytest.ini_options]
addopts = "-v"
testpaths = [
    "tests",
]
# Look for tests in files named test_*.py or check_*.py
python_files = ["test_*.py", "check_*.py"]
# Look for test methods inside classes prefixed with "Test" or "Check"
python_classes = ["Test*", "Check*"]
# Look for test functions prefixed with "test_" or "check_"
python_functions = ["test_*", "check_*"]
```

This gives you complete control over what pytest considers a test.

### `addopts`: Adding More Default Options

The `addopts` key is a space-separated string of command-line arguments you want to run every time. This is perfect for ensuring consistency across all test runs, especially in a team or a CI/CD environment.

Let's add a few more common options:
- `--strict-markers`: Fails the test suite if you use a marker that isn't registered. This prevents typos and keeps your markers clean (more in Chapter 6).
- `-ra`: Shows a short summary for all test outcomes except passes (`r` for report, `a` for all-but-passing). This is great for seeing skips and xfails at a glance.

```toml
# pyproject.toml
[tool.pytest.ini_options]
# Options are space-separated
addopts = "-v -ra --strict-markers"
testpaths = [
    "tests",
]
```

### `markers`: Registering Custom Markers

As we'll see in Chapter 6, markers are a powerful way to categorize tests. The `--strict-markers` option requires you to register them here. Even if you don't use strict mode, registering markers is a best practice as it provides a single source of truth for what each marker means.

```toml
# pyproject.toml
[tool.pytest.ini_options]
addopts = "-v -ra --strict-markers"
testpaths = [
    "tests",
]
markers = [
    "slow: marks tests as slow (deselect with '-m \"not slow\"')",
    "api: marks tests that hit a live API endpoint",
]
```

This configuration now serves as both a functional setting for pytest and documentation for your project's test categories.

## Setting Up Your IDE for Testing

## Bringing Pytest into Your Workflow

Running `pytest` from the command line is the fundamental way to execute your test suite. However, modern Integrated Development Environments (IDEs) like VS Code and PyCharm offer powerful integrations that can make you far more productive. These integrations allow you to run tests, view results, and debug failures directly within your editor.

We'll use Visual Studio Code as our primary example, as it's a popular and free choice. The principles are nearly identical for other IDEs like PyCharm.

### Step 1: Open the Project and Select the Interpreter

First, open your `wallet_project/` folder in VS Code. The IDE needs to know which Python interpreter to use, and it's crucial that it uses the one from your virtual environment.

-   Open the Command Palette (`Ctrl+Shift+P` or `Cmd+Shift+P`).
-   Type "Python: Select Interpreter".
-   Choose the interpreter that includes `.venv` in its path. It will often be labeled with `('.venv')`.

This tells VS Code to use the isolated environment where you installed pytest.

### Step 2: Configure the Test Runner

Next, you need to tell VS Code that you're using pytest.

-   Open the Command Palette again.
-   Type "Python: Configure Tests".
-   Select `pytest` from the list of frameworks.
-   When prompted, select the `tests` directory as the location of your tests.

VS Code will create a `.vscode/settings.json` file in your project to store this configuration. It will look something like this:

```json
{
    "python.testing.pytestArgs": [
        "tests"
    ],
    "python.testing.unittestEnabled": false,
    "python.testing.pytestEnabled": true
}
```

### Step 3: Discover and Run Tests

Once configured, VS Code will automatically discover your tests.

-   Click on the **Testing** icon (a beaker) in the activity bar on the left.
-   You should see a Test Explorer panel appear, showing a tree view of all your test files, classes, and functions.
-   You can now run tests with a single click:
    -   Click the main "play" button at the top of the Test Explorer to run the entire suite.
    -   Hover over a specific file or function to see a "play" button next to it, allowing you to run just that test or file.

You will also see small "Run Test | Debug Test" links appear directly above your test functions in the code editor. This is called a "CodeLens" and is an incredibly fast way to run a single test you are working on.

![VS Code Test Integration](https://code.visualstudio.com/assets/docs/python/testing/images/test-results-in-test-explorer.png)

### The Power of IDE Integration

Why is this so powerful?

-   **Speed:** Running a single test you're focused on is much faster than running the whole suite.
-   **Debugging:** You can click "Debug Test" to run a test with the debugger attached, allowing you to set breakpoints and inspect variables at the exact moment of failure. This transforms debugging from guesswork to a systematic process.
-   **Visual Feedback:** Seeing green checks and red crosses in the UI provides immediate, clear feedback on the health of your code.

Setting up your IDE correctly is a one-time investment that pays off every single time you run a test. It closes the loop between writing code, testing it, and fixing it, making you a faster, more effective developer.
