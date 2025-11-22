# Chapter 16: Continuous Integration and Automation

## Running Tests in CI/CD Pipelines

## The Automation Imperative

So far, you've been running `pytest` on your local machine. This is essential for development, but it has a fundamental flaw: it relies on you, the developer, to remember to run the tests. What happens if you're in a hurry and forget? What if a colleague makes a change and doesn't run the full test suite?

This is where Continuous Integration (CI) comes in.

**Continuous Integration** is the practice of automatically building and testing your code every time a change is pushed to a shared repository. A **CI/CD Pipeline** is the automated workflow that executes these steps.

The core idea is simple but powerful:
1.  A developer pushes a code change to a repository (e.g., on GitHub, GitLab).
2.  This push automatically triggers a process on a server called a "runner" or "agent".
3.  The runner creates a clean, temporary environment.
4.  It checks out the latest code.
5.  It installs the project's dependencies.
6.  It runs the entire test suite using `pytest`.
7.  It reports the result:
    *   **Success:** All tests passed. The change is considered safe to merge.
    *   **Failure:** At least one test failed. The change is flagged as problematic, preventing broken code from being integrated.

This automated process acts as a safety net for your entire team. It ensures that no change can break the existing functionality without immediate detection. It's the foundation of modern, high-quality software development.

In this chapter, we'll move pytest from your local machine into this automated world. We'll explore how to set up CI pipelines on popular platforms to run your tests automatically, providing constant feedback and protecting your codebase. The commands you've learned, like `pytest`, remain the same; we're just changing *who* (or what) runs them.

## GitHub Actions for Python Testing

## GitHub Actions for Python Testing

GitHub Actions is a CI/CD system built directly into GitHub. If your code is hosted there, it's one of the easiest ways to get started with automation. Workflows are defined in YAML files placed in a special directory within your project.

### The Wrong Way: Manual Testing

Imagine you have a simple project with a `requirements.txt` and a `tests/` directory. The manual process is:

1.  `git pull` to get the latest code.
2.  `pip install -r requirements.txt` to update dependencies.
3.  `pytest` to run the tests.

This is fine for one person, but it doesn't scale and is prone to human error. Let's automate it.

### Setting Up Your First Workflow

To create a GitHub Action, you need to create a YAML file in the `.github/workflows/` directory of your repository. Let's call our file `ci.yml`.

First, let's create a simple project structure to test.

**Project Structure:**
```
my_project/
├── .github/
│   └── workflows/
│       └── ci.yml
├── src/
│   └── calculator.py
├── tests/
│   └── test_calculator.py
└── requirements.txt
```

Here's the code for our simple project.

```python
# src/calculator.py
def add(a, b):
    """Adds two numbers together."""
    if not isinstance(a, (int, float)) or not isinstance(b, (int, float)):
        raise TypeError("Both arguments must be numbers")
    return a + b
```

```python
# tests/test_calculator.py
import pytest
from src.calculator import add

def test_add_integers():
    assert add(2, 3) == 5

def test_add_floats():
    assert add(2.5, 3.5) == 6.0

def test_add_raises_type_error_for_strings():
    with pytest.raises(TypeError):
        add("two", "three")
```

```text
# requirements.txt
pytest
```

Now, let's create the GitHub Actions workflow file. This file tells GitHub what to do when code is pushed.

```yaml
# .github/workflows/ci.yml

# A name for the workflow, which will be displayed on GitHub
name: Python CI

# Trigger the workflow on pushes and pull requests to the 'main' branch
on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

# Define the jobs to be run. We only have one job here: 'build'.
jobs:
  build:
    # The type of runner that the job will run on.
    # 'ubuntu-latest' is a good default.
    runs-on: ubuntu-latest

    # A job is a sequence of steps.
    steps:
      # Step 1: Check out the repository code so the runner can access it.
      - name: Check out repository
        uses: actions/checkout@v4

      # Step 2: Set up a specific version of Python.
      - name: Set up Python 3.10
        uses: actions/setup-python@v5
        with:
          python-version: "3.10"

      # Step 3: Install project dependencies.
      # This is the same command you'd run locally.
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      # Step 4: Run pytest.
      # The core of our CI pipeline!
      - name: Run tests with pytest
        run: pytest
```

### How It Works

Let's break down the YAML file:

-   **`name`**: The name of your workflow, shown in the "Actions" tab on GitHub.
-   **`on`**: Defines the events that trigger this workflow. Here, it runs on any `push` or `pull_request` to the `main` branch.
-   **`jobs`**: A workflow is made up of one or more jobs. Our job is named `build`.
-   **`runs-on`**: Specifies the virtual environment to run the job in. `ubuntu-latest` is a common and reliable choice.
-   **`steps`**: This is the sequence of tasks the job will execute.
    -   `uses: actions/checkout@v4`: This is a pre-built action that downloads your repository's code into the runner.
    -   `uses: actions/setup-python@v5`: Another pre-built action that installs a specific Python version.
    -   `run: ...`: This executes shell commands. We use it to install our dependencies from `requirements.txt` and then to run `pytest`.

Once you commit and push this file, go to the "Actions" tab of your repository on GitHub. You'll see your workflow running automatically!

### Testing on Multiple Python Versions

What if your project needs to support Python 3.9, 3.10, and 3.11? Running tests on all versions manually is tedious. With CI, it's trivial. We can use a `strategy` matrix to run the same job multiple times with different Python versions.

```yaml
# .github/workflows/ci.yml (updated with a matrix)

name: Python CI

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    
    # Define a matrix strategy
    strategy:
      # Don't cancel all jobs if one fails
      fail-fast: false
      matrix:
        # List the Python versions to test against
        python-version: ["3.9", "3.10", "3.11"]

    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      # Use the python-version from the matrix
      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run tests with pytest
        run: pytest
```

With this change, GitHub will now automatically start three separate jobs in parallel, one for each Python version specified in the `matrix`. The `${{ matrix.python-version }}` syntax is how you access the current value from the matrix within your steps.

This is a massive leap in testing robustness. You can be confident that your code works across all supported Python versions with every single change you make.

## GitLab CI Integration

## GitLab CI Integration

GitLab has its own powerful, integrated CI/CD system. The concepts are identical to GitHub Actions—automating setup and test execution—but the syntax is different. Workflows are defined in a single file named `.gitlab-ci.yml` at the root of your repository.

### The `.gitlab-ci.yml` File

GitLab CI is built around the concepts of **stages** and **jobs**. A typical pipeline might have stages like `build`, `test`, and `deploy`. Jobs are assigned to stages and run in order.

Let's create a `.gitlab-ci.yml` for the same calculator project.

```yaml
# .gitlab-ci.yml

# Define the stages for the pipeline. Jobs will run in this order.
stages:
  - test

# Use a Docker image that comes with Python 3.10 pre-installed.
# This is the environment where our commands will run.
image: python:3.10

# This block of commands runs before each job.
# It's a great place for setup tasks like installing dependencies.
before_script:
  - python -m pip install --upgrade pip
  - pip install -r requirements.txt

# Define a job named 'pytest_job'. It belongs to the 'test' stage.
pytest_job:
  stage: test
  # The actual commands to execute for this job.
  script:
    - pytest
```

### How It Works

-   **`stages`**: We define a single stage named `test`.
-   **`image`**: This is a key concept in GitLab CI. It specifies a Docker image to use for the job's environment. By using `python:3.10`, we get a clean environment with Python 3.10 ready to go, saving us a setup step.
-   **`before_script`**: These commands are executed before the `script` section of any job. It's the perfect place to install dependencies, ensuring they are available for the test run.
-   **`pytest_job`**: This is the name of our job.
    -   `stage: test`: Assigns this job to the `test` stage.
    -   `script`: The list of shell commands to execute. Here, we simply run `pytest`.

When you push this file to a GitLab repository, the pipeline will automatically start, create a Docker container from the `python:3.10` image, run the `before_script` commands, and finally execute the `pytest` command.

### Testing on Multiple Python Versions in GitLab

Just like with GitHub Actions, we can test against multiple Python versions using a matrix.

```yaml
# .gitlab-ci.yml (updated with a matrix)

stages:
  - test

# We don't define a global image anymore, as each job will specify its own.

pytest_job:
  stage: test
  # The script is the same for all matrix jobs.
  script:
    - python -m pip install --upgrade pip
    - pip install -r requirements.txt
    - pytest
  
  # Define the matrix strategy
  parallel:
    matrix:
      # Define a variable named PYTHON_VERSION with a list of values.
      - PYTHON_VERSION: ["3.9", "3.10", "3.11"]

  # Use the matrix variable to select the Docker image for each job.
  image: python:${PYTHON_VERSION}
```

Here's the updated logic:

-   We removed the global `image` and `before_script`.
-   The `pytest_job` now has a `parallel: matrix` section. This tells GitLab to create multiple parallel jobs based on the variables defined.
-   We define a variable `PYTHON_VERSION` with our target versions.
-   In the `image` key for the job, we use `python:${PYTHON_VERSION}`. GitLab substitutes this variable for each job, so it will use `image: python:3.9`, `image: python:3.10`, and so on.
-   The setup commands are now moved inside the `script` block, as they need to run in each of the parallel jobs.

The principle is the same as with GitHub Actions: define your variations in a matrix and use a variable to apply that variation to your job's configuration. The result is a robust test suite that runs automatically across all your supported environments.

## Jenkins and Other CI Systems

## Jenkins and Other CI Systems

While GitHub Actions and GitLab CI are popular choices for projects hosted on their respective platforms, many other CI/CD systems exist, such as Jenkins, CircleCI, Travis CI, and Bitbucket Pipelines. Jenkins is one of the oldest and most powerful, often used for complex, self-hosted enterprise workflows.

The good news is that the core principles are universal. No matter the system, you will always perform these steps:
1.  Define a trigger (e.g., on commit).
2.  Specify an execution environment (e.g., a Docker container, a specific virtual machine).
3.  Check out the code.
4.  Run shell commands to install dependencies.
5.  Run shell commands to execute tests.

The only thing that changes is the syntax of the configuration file.

### Jenkins and the `Jenkinsfile`

Jenkins pipelines are typically defined in a file named `Jenkinsfile` in the root of the repository. This is called "pipeline-as-code."

Here is a simple declarative `Jenkinsfile` that accomplishes the same task as our previous examples. This example assumes Jenkins is configured to run jobs inside a Docker container.

```groovy
// Jenkinsfile

pipeline {
    // Define the agent (execution environment) for the entire pipeline.
    // Here, we use a Docker container with Python 3.10.
    agent {
        docker { image 'python:3.10' }
    }

    // Define the stages of the pipeline.
    stages {
        // The 'Test' stage
        stage('Test') {
            // The steps to execute in this stage.
            steps {
                // The 'sh' step executes a shell command.
                sh 'python -m pip install --upgrade pip'
                sh 'pip install -r requirements.txt'
                sh 'pytest'
            }
        }
    }
}
```

### How It Works

-   **`pipeline`**: The top-level block that defines the entire workflow.
-   **`agent`**: Specifies where the pipeline will run. Using `agent { docker { ... } }` is a modern best practice, ensuring a clean, reproducible environment for every run, just like with GitLab CI.
-   **`stages`**: A container for one or more `stage` blocks.
-   **`stage('Test')`**: Defines a logical stage of the pipeline. In the Jenkins UI, you'll see the progress of each stage.
-   **`steps`**: The actual work happens here.
-   **`sh '...'`**: The command to execute a shell script. This is where we run our familiar `pip` and `pytest` commands.

### The Universal Takeaway

Don't get bogged down in the specific syntax of each platform. Instead, focus on the pattern. Your `pytest` command is the heart of the operation. The CI system is just a wrapper—an automated shell—that prepares an environment and runs that command for you.

If you need to use a new CI system, your task is to look up its documentation for the following:
1.  How do I specify a Python environment (or Docker image)?
2.  How do I check out code?
3.  How do I run a shell command?

Once you know those three things, you can run your pytest suite anywhere.

## Generating Test Reports

## Generating Test Reports

When a CI pipeline fails, the log output tells you what went wrong. However, scrolling through hundreds of lines of text to find the error can be inefficient. Most CI systems can parse structured test reports to provide a much better user interface, showing a summary of test results, highlighting failures, and tracking test duration.

The two most common report formats are **JUnit XML** and **HTML**.

### JUnit XML Reports for CI Integration

JUnit is a testing framework for Java, but its XML report format has become a de facto standard for CI/CD systems. Pytest can generate these reports easily.

To generate a JUnit XML report, use the `--junitxml` command-line flag.

Let's modify our CI scripts to generate and save this report.

```bash
# Command to run in your CI script
pytest --junitxml=report.xml
```

This command runs your tests as usual, but it also creates a file named `report.xml` with the structured results.

The next step is to tell the CI system that this file is an "artifact"—a file that should be saved and associated with the pipeline run.

#### GitHub Actions Example with Reports

```yaml
# .github/workflows/ci.yml (with reporting)

# ... (name, on, jobs, runs-on, strategy sections are the same) ...

    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: Run tests with pytest
        run: pytest --junitxml=report.xml

      # New Step: Upload the test report as an artifact
      - name: Upload test report
        # This condition ensures the step runs even if previous steps fail
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: pytest-report-${{ matrix.python-version }}
          path: report.xml
```

The new `Upload test report` step uses the `actions/upload-artifact` action. After the workflow run is complete, you can download the `report.xml` file from the workflow summary page. Many GitHub Apps can also parse these reports to provide richer feedback directly in pull requests.

#### GitLab CI Example with Reports

```yaml
# .gitlab-ci.yml (with reporting)

# ... (stages, etc. are the same) ...

pytest_job:
  stage: test
  script:
    - pip install -r requirements.txt
    - pytest --junitxml=report.xml
  
  # New Section: Define artifacts
  artifacts:
    # This condition ensures artifacts are saved even if the job fails
    when: always
    paths:
      - report.xml
    # Specify the report type for better UI integration
    reports:
      junit: report.xml
```

GitLab has first-class support for JUnit reports. By adding the `artifacts` section and specifying `reports: junit`, GitLab will automatically parse the `report.xml` file. In a merge request, it will show a summary of test changes, telling you which tests started failing, which were fixed, and which are new. This is incredibly valuable for code review.

### HTML Reports for Human-Readable Output

While JUnit XML is great for machines, it's not easy for humans to read. For a visually appealing, shareable report, you can use the `pytest-html` plugin.

First, add it to your `requirements.txt`.

```text
# requirements.txt
pytest
pytest-html
```

Next, update your test command to generate an HTML report.

```bash
# Command to generate an HTML report
pytest --html=report.html --self-contained-html
```

The `--self-contained-html` flag is useful for CI because it embeds the CSS into the HTML file, making it a single, portable file.

You can then configure your CI pipeline to save this `report.html` as an artifact, just as we did with the XML file. This gives you a downloadable, browser-viewable report for every test run, which is excellent for manual inspection and archiving.

## Automated Testing on Multiple Python Versions (tox)

## Automated Testing on Multiple Python Versions (tox)

We've seen how to configure CI platforms to run tests against a matrix of Python versions. This works, but it has a downside: the logic for installing dependencies and running tests is defined in the CI configuration file (e.g., `ci.yml`). If you want to run the same multi-version tests locally, you have to manually replicate those steps.

This is the problem that `tox` solves. **Tox is a command-line tool that automates and standardizes testing in Python.** It allows you to define your test environments and commands in a single `tox.ini` file.

### The Pain Before Tox

To test your project locally against Python 3.9 and 3.10, you would have to:
1.  Make sure you have both Python 3.9 and 3.10 installed.
2.  Create a virtual environment with Python 3.9.
3.  Activate it and run `pip install -r requirements.txt`.
4.  Run `pytest`.
5.  Deactivate and delete the virtual environment.
6.  Repeat steps 2-5 for Python 3.10.

This is tedious and error-prone. `tox` automates this entire process.

### Setting Up `tox`

First, install `tox` and add it to your requirements (or a `requirements-dev.txt`).

```bash
pip install tox
```

Next, create a `tox.ini` file in your project's root directory.

```ini
# tox.ini

[tox]
# Define the environments to run by default.
# 'py39' means "run with Python 3.9".
envlist = py39, py310, py311

[testenv]
# Define dependencies needed for testing.
# This is separate from your main requirements.txt.
deps =
    pytest
    pytest-html

# Define the command(s) to run for testing.
commands =
    pytest --junitxml=report.xml --html=report.html
```

### How It Works

-   **`[tox]` section**:
    -   `envlist`: This is the list of environments `tox` will manage. When you run the `tox` command, it will execute the test suite for each environment in this list: `py39` (Python 3.9), `py310` (Python 3.10), and `py311` (Python 3.11).
-   **`[testenv]` section**:
    -   This is a template for each environment defined in `envlist`.
    -   `deps`: A list of dependencies to install into the temporary virtual environment that `tox` creates.
    -   `commands`: The commands to execute within that environment.

Now, from your terminal, you can simply run:

```bash
tox
```

`tox` will perform the following actions automatically:
1.  Look for `python3.9` on your system. If found, it creates a new virtual environment.
2.  Installs `pytest` and `pytest-html` into that environment.
3.  Runs the `pytest ...` command.
4.  Repeats the process for `python3.10` and `python3.11`.
5.  Reports a summary of which environments passed or failed.

### Integrating `tox` with CI

The real power of `tox` comes from simplifying your CI configuration. Instead of telling your CI system *how* to test your code, you just tell it to run `tox`. The `tox.ini` file becomes the single source of truth for your testing procedure.

Here is our GitHub Actions workflow, refactored to use `tox`.

```yaml
# .github/workflows/ci.yml (refactored with tox)

name: Python CI with tox

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        python-version: ["3.9", "3.10", "3.11"]

    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Set up Python ${{ matrix.python-version }}
        uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}

      # Install tox
      - name: Install tox
        run: pip install tox

      # Run tox. Tox will select the correct environment based on the Python version.
      - name: Run tests with tox
        run: tox
```

Look how much simpler the testing logic is!

-   The `Install dependencies` step is gone.
-   The `Run tests with pytest` step is replaced with a simple `run: tox`.

The CI runner sets up a specific Python version (e.g., 3.9), and when `tox` runs, it intelligently detects that it's running in a Python 3.9 environment and therefore only executes the `py39` test environment from `tox.ini`.

This pattern is a professional best practice:
-   **Single Source of Truth**: The `tox.ini` file defines your test procedure for both local development and CI.
-   **Reproducibility**: Any developer can clone the repository, run `tox`, and get the exact same test results as the CI server.
-   **Simplicity**: Your CI configuration becomes minimal and declarative. It's only responsible for setting up environments, not for defining test logic.
