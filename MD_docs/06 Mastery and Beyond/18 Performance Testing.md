# Chapter 18: Performance Testing

## When Performance Testing Matters

## When Performance Testing Matters

So far, our focus has been on correctness: does our code produce the right output? But in the real world, another question is just as critical: is our code *fast enough*? A correct algorithm that takes ten minutes to sort a list of a thousand items is practically useless. A web page that takes 30 seconds to load will be abandoned by users.

Performance testing isn't about making every line of code as fast as possible. That's a path to premature optimization and unreadable code. Instead, it's about identifying and protecting the performance of the critical paths in your application.

### What is a "Critical Path"?

A critical path is a sequence of operations or a piece of code whose performance has a direct and significant impact on the user experience or system efficiency. Think about:

-   **Core Algorithms:** The sorting function in your e-commerce backend that ranks products by relevance.
-   **Data Processing Pipelines:** An ETL job that processes millions of log entries per hour.
-   **API Endpoints:** The `/api/v1/search` endpoint that is hit thousands of times per minute.
-   **Resource-Intensive Tasks:** Image processing, video encoding, or complex scientific calculations.

For these parts of your system, a small performance degradation can have a massive ripple effect, leading to higher server costs, slower response times, and unhappy users.

### The Goal: Preventing Regressions

The primary goal of performance testing within your test suite is not to achieve the absolute fastest speed, but to **prevent performance regressions**. A regression is when a code change inadvertently makes a critical path slower.

Imagine a developer makes a seemingly harmless change to a utility function. They run the correctness tests, everything passes, and they merge the code. A week later, customers start complaining that the search feature is sluggish. The team scrambles to find the cause, eventually tracing it back to that "harmless" change which, it turns out, is called by the search algorithm and has introduced a 200ms delay.

Performance tests act as a safety net. They establish a baseline for how your critical code *should* perform and automatically flag any new code that deviates significantly from that baseline.

### Types of Performance Testing

In the context of pytest, we'll focus on two main types:

1.  **Micro-benchmarking:** Measuring the execution speed of a single, small unit of code, like a function or a method. This is excellent for optimizing specific algorithms.
2.  **Macro-benchmarking:** Measuring the performance of a larger operation, like a full API request-response cycle or processing a file. This is more representative of the user experience.

Throughout this chapter, we'll learn the tools to implement both, moving from simple, naive measurements to robust, statistically sound benchmarking that can be integrated directly into your development workflow.

## Measuring Test Execution Time

## Measuring Test Execution Time

Before we dive into specialized tools, let's explore the most basic ways to measure time and understand their limitations. This will illuminate the path by first showing the pits, making the case for why more robust tools are necessary.

### The Wrong Way: `time.time()`

A beginner's first instinct might be to use Python's built-in `time` module directly within a test.

Let's imagine we have a simple function that sorts a list of numbers. We want to ensure it completes within a certain timeframe.

```python
# src/sorting.py
import time

def bubble_sort(items):
    """A deliberately inefficient sorting algorithm."""
    n = len(items)
    for i in range(n):
        for j in range(0, n - i - 1):
            if items[j] > items[j + 1]:
                items[j], items[j + 1] = items[j + 1], items[j]
    return items

def fast_sort(items):
    """A much more efficient sorting algorithm."""
    return sorted(items)
```

Now, let's write a test using `time.time()`.

```python
# tests/test_sorting_perf_naive.py
import time
import random
from src.sorting import bubble_sort

def test_bubble_sort_performance_naive():
    # Setup
    items = [random.randint(1, 1000) for _ in range(500)]
    
    # Measure
    start_time = time.time()
    bubble_sort(items)
    end_time = time.time()
    
    duration = end_time - start_time
    print(f"\nBubble sort took {duration:.4f} seconds")
    
    # Assert
    assert duration < 0.05 # 50 milliseconds
```

If you run this test, it will likely fail.

```bash
$ pytest -v -s tests/test_sorting_perf_naive.py
=========================== test session starts ============================
...
collected 1 item

tests/test_sorting_perf_naive.py::test_bubble_sort_performance_naive 
Bubble sort took 0.0612 seconds
FAILED

================================= FAILURES =================================
___________ test_bubble_sort_performance_naive ___________

    def test_bubble_sort_performance_naive():
        # Setup
        items = [random.randint(1, 1000) for _ in range(500)]
        
        # Measure
        start_time = time.time()
        bubble_sort(items)
        end_time = time.time()
        
        duration = end_time - start_time
        print(f"\nBubble sort took {duration:.4f} seconds")
        
        # Assert
>       assert duration < 0.05 # 50 milliseconds
E       assert 0.0612... < 0.05

tests/test_sorting_perf_naive.py:17: AssertionError
========================= 1 failed in ...s =========================
```

This approach is fundamentally flawed for several reasons:
1.  **It's Unstable:** The execution time depends heavily on what else your computer is doing. If your OS decides to run a background process, the test will slow down and fail. This leads to "flaky" tests.
2.  **It's Not Statistically Significant:** A single run is not a reliable measure. A proper benchmark needs to run the code many times to get a stable average and understand the variance.
3.  **It's Not Portable:** A test that passes on a powerful developer machine might fail on a slower CI server. The hardcoded threshold (`0.05`) is arbitrary and brittle.

This method tells you very little, and what it does tell you is unreliable.

### The Pytest Way: Finding Slow Tests with `--durations`

Pytest has a built-in feature that is perfect for identifying which *tests* in your suite are taking the longest to run. It's not a benchmarking tool for your *code*, but an invaluable diagnostic tool for your *test suite*.

The `--durations` option reports the N slowest test durations.

Let's add a few more tests to see it in action.

```python
# tests/test_suite_speed.py
import time
import random
from src.sorting import bubble_sort, fast_sort

def test_fast_sort():
    items = [random.randint(1, 1000) for _ in range(500)]
    fast_sort(items)
    assert True

def test_bubble_sort():
    items = [random.randint(1, 1000) for _ in range(500)]
    bubble_sort(items)
    assert True

def test_quick_check():
    time.sleep(0.01)
    assert True
```

Now, run pytest with `--durations=3`:

```bash
$ pytest --durations=3 tests/test_suite_speed.py
=========================== test session starts ============================
...
collected 3 items

tests/test_suite_speed.py ...                                        [100%]

======================== slowest 3 test durations ========================
0.06s call     tests/test_suite_speed.py::test_bubble_sort
0.01s call     tests/test_suite_speed.py::test_quick_check
0.00s call     tests/test_suite_speed.py::test_fast_sort
========================= 3 passed in ...s =========================
```

The output clearly shows that `test_bubble_sort` is by far the slowest test. This is the correct tool for the job of monitoring your test suite's health. If you see a test suddenly appear at the top of this list after a code change, it's a strong signal that you've introduced a performance issue in your tests.

However, it still doesn't solve our original problem: how to reliably benchmark a specific piece of code and prevent regressions. For that, we need a dedicated plugin.

## pytest-benchmark for Reliable Benchmarks

## pytest-benchmark for Reliable Benchmarks

To overcome the limitations of naive timing, we need a tool that performs statistically sound measurements. The most popular and powerful tool for this in the pytest ecosystem is `pytest-benchmark`.

It solves the problems we identified:
-   **Reliability:** It runs your code many times in a loop to get a stable average, minimizing the impact of system noise.
-   **Statistical Rigor:** It provides rich data, including minimum, maximum, mean, and standard deviation.
-   **Regression Tracking:** It can save results from a run and compare them against future runs, automatically detecting performance changes.

### Installation

First, install the plugin into your virtual environment.

```bash
pip install pytest-benchmark
```

### Your First Benchmark

Using `pytest-benchmark` is incredibly simple. It provides a special `benchmark` fixture. You pass the function you want to test to this fixture.

Let's rewrite our sorting test the right way.

```python
# tests/test_sorting_benchmark.py
import random
from src.sorting import bubble_sort, fast_sort

# Prepare some data to be used by the tests
random_data = [random.randint(1, 1000) for _ in range(500)]

def test_bubble_sort_benchmark(benchmark):
    # The benchmark fixture receives the function to test
    # The lambda is used to pass arguments to the function
    result = benchmark(lambda: bubble_sort(random_data.copy()))
    
    # You can still assert correctness
    assert result == sorted(random_data)

def test_fast_sort_benchmark(benchmark):
    result = benchmark(lambda: fast_sort(random_data.copy()))
    assert result == sorted(random_data)
```

Note that we pass a `lambda` to `benchmark()`. This is how you call a function that requires arguments. We also use `random_data.copy()` to ensure each run of the benchmark gets a fresh, unsorted list.

Now, run pytest. The output is much more informative.

```bash
$ pytest tests/test_sorting_benchmark.py
=========================== test session starts ============================
...
plugins: benchmark-4.0.0
...
collected 2 items

tests/test_sorting_benchmark.py::test_bubble_sort_benchmark ✓         [ 50%]
tests/test_sorting_benchmark.py::test_fast_sort_benchmark ✓         [100%]

---------------------------------------------------------------------------------- benchmark: 2 tests ----------------------------------------------------------------------------------
Name (time in ms)                       Min                 Max                Mean             StdDev              Median               IQR            Outliers     OPS            Rounds
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
test_fast_sort_benchmark           0.0046 (1.0)        0.0183 (1.0)        0.0056 (1.0)       0.0018 (1.0)        0.0051 (1.0)        0.0005 (1.0)       103;140  179,245.8055 (1.0)      1000
test_bubble_sort_benchmark        51.3411 (11,221.9)  59.9411 (3,275.6)   53.5981 (9,613.8)   2.2091 (1,227.3)    52.9821 (10,413.0)   2.2998 (4,599.6)      2;2    18.6574 (0.0001)        5
------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Legend:
  Outliers: 1 Standard Deviation from Mean; 1.5 IQR (InterQuartile Range) from 1st Quartile and 3rd Quartile.
  OPS: Operations Per Second, computed as 1 / Mean.
============================ 2 passed in ...s ============================
```

### Reading the Benchmark Report

Let's break down this table. `pytest-benchmark` ran `fast_sort` 1000 times (`Rounds`) and `bubble_sort` only 5 times. It automatically calibrates the number of rounds to get a statistically stable result within a reasonable amount of time.

-   **Min/Max/Mean:** The minimum, maximum, and average time for a single execution. The `Mean` is usually the most important number.
-   **StdDev:** The standard deviation, which tells you how much the timing varied between runs. A low StdDev is good.
-   **OPS:** Operations Per Second. This is simply `1 / Mean`, a useful metric for throughput.
-   **Comparison numbers `(...)`:** The numbers in parentheses show the performance ratio compared to the fastest test. Here, `bubble_sort`'s mean time is over 9,600 times slower than `fast_sort`.

### Detecting Regressions

This report is useful, but the real power comes from comparing runs over time.

First, let's save a baseline result.

```bash
# Save the current results to a file in .benchmarks/
pytest --benchmark-save=sorting_baseline
```

Now, let's introduce a performance regression. We'll add an unnecessary `time.sleep()` to our "fast" sort function.

```python
# src/sorting.py (modified)
import time

# ... bubble_sort remains the same ...

def fast_sort(items):
    """A much more efficient sorting algorithm... with a regression."""
    time.sleep(0.001) # Simulate doing extra, slow work
    return sorted(items)
```

Now, run the benchmarks again, but this time, compare them to the baseline we just saved.

```bash
$ pytest --benchmark-compare=sorting_baseline
...
------------------------------------------------------------------------------------ benchmark: 2 tests ------------------------------------------------------------------------------------
Name (time in ms)                       Min                 Max                Mean             StdDev              Median               IQR            Outliers     OPS            Rounds
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
test_fast_sort_benchmark           1.0118 (221.1)      1.2121 (66.2)       1.0321 (185.1)      0.0321 (17.8)       1.0249 (201.4)      0.0328 (65.6)        5;8     968.8693 (0.005)       959
test_bubble_sort_benchmark        51.5805 (1.0)       55.9805 (0.9)       52.9812 (1.0)        1.3521 (0.6)       52.6811 (1.0)        1.6011 (0.7)         2;2      18.8746 (1.0)          5
--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

Benchmark comparison appears to have degraded performance.
  1/2 benchmarks faster (50.00%)
  1/2 benchmarks slower (50.00%)
  0/2 benchmarks unchanged (0.00%)
ERROR: Benchmark performance has degraded.
=========================== 2 passed, 1 error in ...s ============================
```

Pytest exits with an error! The report shows that `test_fast_sort_benchmark` is now ~185 times slower than the baseline (`Mean` column). `pytest-benchmark` has successfully caught our performance regression automatically. This is the core workflow for performance testing.

## Memory Profiling in Tests

## Memory Profiling in Tests

Performance isn't just about speed; it's also about memory. A function that is lightning fast but consumes gigabytes of RAM for a simple task can be just as problematic as a slow one, especially in data science applications or long-running services where memory leaks can be catastrophic.

While `pytest-benchmark` is the king of timing, we need a different tool for memory. A fantastic modern option is `pytest-memray`. `Memray` is a memory profiler for Python that can track every allocation, helping you find memory leaks and identify code that allocates too much memory.

### Installation

Install `memray` and its pytest plugin.

```bash
pip install memray pytest-memray
```

### A Memory-Hungry Example

Let's write a function that is inefficient in its memory usage. This function will build a list of a million strings, but it does so by creating a large intermediate list of numbers first, which is unnecessary.

```python
# src/memory_hog.py

def process_data_inefficiently(size=1_000_000):
    """Creates a list of strings, but via a large intermediate list."""
    # This intermediate list consumes a lot of memory
    intermediate_numbers = list(range(size))
    
    # The final list also consumes memory
    final_strings = [str(i) for i in intermediate_numbers]
    return final_strings

def process_data_efficiently(size=1_000_000):
    """Creates the same list of strings using a memory-efficient generator."""
    # A generator doesn't create the intermediate list in memory
    final_strings = [str(i) for i in range(size)]
    return final_strings
```

Now, let's write a simple test for one of these functions. We don't need any special fixtures; we just need to run pytest with the right command-line flag.

```python
# tests/test_memory.py
from src.memory_hog import process_data_inefficiently

def test_inefficient_processing():
    result = process_data_inefficiently()
    assert len(result) == 1_000_000
    assert result[123] == "123"
```

### Running the Memory Profiler

To profile this test, we use the `--memray` flag.

```bash
pytest --memray tests/test_memory.py
```

This command will run the test and, upon completion, generate a detailed report file (e.g., `memray-test_inefficient_processing.bin`). The output will tell you how to view the results.

```bash
...
[100%] PASSED
A memory profile has been generated.

To view the flame graph, run:
 python3 -m memray flamegraph .../memray-test_inefficient_processing.bin
```

### Interpreting the Report

The most intuitive way to view a `memray` report is as a "flame graph." Running the command suggested in the output will generate an HTML file and open it in your browser.

A flame graph is a visualization of your program's memory allocations.
-   The **width** of a bar represents the proportion of total memory allocated by that function and its children.
-   The **y-axis** represents the call stack. Functions at the bottom call the functions directly above them.

When you analyze the flame graph for `test_inefficient_processing`, you will see a very wide bar corresponding to the line `intermediate_numbers = list(range(size))`. This immediately tells you that this specific line of code is responsible for a huge chunk of the memory allocation during the test.

If you were to profile `process_data_efficiently` instead, you would see a much smaller memory footprint, as the generator expression avoids creating the large intermediate list.

Using `pytest-memray` allows you to pinpoint the exact lines of code that are causing high memory usage, making it an essential tool for optimizing memory-intensive applications.

## Identifying and Fixing Slow Tests

## Identifying and Fixing Slow Tests

So far, we've focused on testing the performance of your application code. But what about the performance of your *test suite* itself? A slow test suite is a major drag on developer productivity. If running tests takes 20 minutes, developers will run them less often, feedback loops will lengthen, and the benefits of rapid testing will be lost.

This section is about using pytest's tools to find and fix the bottlenecks *within your tests*.

### The Tool for the Job: `--durations` Revisited

As we saw in section 18.2, the `pytest --durations=N` flag is the primary tool for this task. It doesn't have the statistical rigor of `pytest-benchmark`, but that's not its purpose. Its goal is to quickly point out the slowest parts of your test suite's execution.

A common practice is to always run it with a small number, like `--durations=10`, to keep an eye on the slowest tests in your project.

### Common Causes of Slow Tests (and How to Fix Them)

Slow tests are rarely caused by slow application code. More often, they are caused by inefficient test setup and teardown. Let's look at the most common culprits.

#### Culprit 1: Inefficient Fixture Scope

This is the most frequent cause of slow test suites. A fixture that performs an expensive operation (like creating a database connection, reading a large file, or setting up a complex object) with the default `function` scope will repeat that expensive operation for *every single test* that uses it.

**The Pit (Wrong Way):** Imagine a fixture that sets up a test database by loading a 100MB SQL dump file.

```python
# tests/conftest.py
import pytest
import time

@pytest.fixture
def db_connection():
    # This is a function-scoped fixture by default
    print("\nSetting up the database (SLOW OPERATION)...")
    time.sleep(2) # Simulate loading a large file
    db = {"user": "test", "data": [1, 2, 3]}
    yield db
    print("\nTearing down the database...")

# tests/test_db_operations.py
def test_read_from_db(db_connection):
    assert db_connection["user"] == "test"

def test_write_to_db(db_connection):
    db_connection["data"].append(4)
    assert 4 in db_connection["data"]

def test_another_check(db_connection):
    assert len(db_connection["data"]) > 0
```

Let's run this and see the durations.

```bash
$ pytest -s --durations=3 tests/test_db_operations.py
=========================== test session starts ============================
...
collected 3 items

tests/test_db_operations.py::test_read_from_db 
Setting up the database (SLOW OPERATION)...
PASSED
Tearing down the database...

tests/test_db_operations.py::test_write_to_db 
Setting up the database (SLOW OPERATION)...
PASSED
Tearing down the database...

tests/test_db_operations.py::test_another_check 
Setting up the database (SLOW OPERATION)...
PASSED
Tearing down the database...

======================== slowest 3 test durations ========================
2.00s call     tests/test_db_operations.py::test_read_from_db
2.00s call     tests/test_db_operations.py::test_write_to_db
2.00s call     tests/test_db_operations.py::test_another_check
========================= 3 passed in ...s =========================
```

Each test took 2 seconds, for a total of 6 seconds. The expensive setup ran three times.

**The Fix:** If the tests don't modify the state of the resource in a way that would affect other tests, you can change the fixture's scope. By changing the scope to `module`, the setup will run only *once* for all tests in that file.

```python
# tests/conftest.py (fixed)
import pytest
import time

@pytest.fixture(scope="module") # Changed scope!
def db_connection():
    print("\nSetting up the database ONCE for the module...")
    time.sleep(2) # Simulate loading a large file
    db = {"user": "test", "data": [1, 2, 3]}
    yield db
    print("\nTearing down the database ONCE...")
```

Now, let's re-run the tests.

```bash
$ pytest -s --durations=3 tests/test_db_operations.py
=========================== test session starts ============================
...
collected 3 items

tests/test_db_operations.py::test_read_from_db 
Setting up the database ONCE for the module...
PASSED
tests/test_db_operations.py::test_write_to_db PASSED
tests/test_db_operations.py::test_another_check PASSED
Tearing down the database ONCE...

======================== slowest 3 test durations ========================
2.00s setup    tests/test_db_operations.py::test_read_from_db
0.00s call     tests/test_db_operations.py::test_read_from_db
0.00s call     tests/test_db_operations.py::test_write_to_db
(setup duration is only reported for the first test that uses the fixture)
========================= 3 passed in ...s =========================
```

The total execution time is now just over 2 seconds instead of 6! The `setup` part took 2 seconds, but the `call` for each test was instantaneous. This is a massive improvement and a critical optimization for any large test suite.

#### Culprit 2: Unnecessary I/O (Network or Disk)

Tests that make real network requests or frequently read/write from the disk will be slow and unreliable.
-   **The Fix:** Mock external services and file system interactions. Use libraries like `pytest-mock` (covered in Chapter 8) to replace slow I/O calls with fast, in-memory fakes. Use the built-in `tmp_path` fixture (Chapter 12) for tests that absolutely must interact with the filesystem.

#### Culprit 3: `time.sleep()`

Explicit sleeps are a major anti-pattern in tests. They are usually added to wait for an asynchronous operation to complete.
-   **The Fix:** Never use `time.sleep()`. If you are testing asynchronous code, use a proper async testing library like `pytest-asyncio` (Chapter 11) that can `await` results. If you are waiting for a resource, implement a polling mechanism with a timeout instead of a fixed sleep.

## Performance Testing in CI/CD

## Performance Testing in CI/CD

Identifying performance issues on your local machine is good, but integrating performance testing into your Continuous Integration/Continuous Deployment (CI/CD) pipeline is where you build a true safety net for your project. This allows you to automatically catch regressions before they ever reach production.

### Strategy 1: Monitor Test Suite Health

The simplest strategy is to monitor the overall speed of your test suite. A sudden increase in total test time can indicate a problem.

You can add a step to your CI workflow that always reports the slowest tests. This doesn't fail the build, but it creates a record and makes performance degradation visible to the whole team.

Here is an example snippet for a GitHub Actions workflow:

```yaml
# .github/workflows/ci.yml
name: Python CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      # ... (checkout code, setup python, install dependencies) ...

      - name: Run tests
        run: pytest

      - name: Report slowest tests
        # This step runs even if the previous one fails
        if: always() 
        run: pytest --durations=20
```

This ensures that every pull request and merge to `main` will include a report of the 20 slowest tests. If a developer sees their new test at the top of that list, it's a clear signal to investigate.

### Strategy 2: Automated Regression Detection with `pytest-benchmark`

This is the most powerful approach. The goal is to automatically fail a CI build if a change introduces a significant performance regression in a critical code path.

The workflow is as follows:
1.  **Establish a Baseline:** Run the benchmarks on your main branch (`main` or `master`) and save the results as a CI artifact. This artifact represents the "known good" performance.
2.  **Compare on Pull Requests:** When a pull request is opened, run the benchmarks again. Use `pytest-benchmark`'s comparison feature to check the new results against the baseline downloaded from the main branch.
3.  **Fail on Regression:** Configure `pytest-benchmark` to fail the build if performance degrades by more than a set threshold.

Here's a conceptual GitHub Actions workflow demonstrating this:

```yaml
# .github/workflows/ci.yml
name: Python CI with Performance Benchmarks

on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]

jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.10'
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
          pip install pytest-benchmark

      - name: Download baseline benchmark data
        # Only run on PRs, not on main branch itself
        if: github.event_name == 'pull_request'
        uses: actions/cache@v3
        with:
          path: .benchmarks
          # Try to get cache from the target branch (main)
          key: benchmark-cache-${{ github.base_ref }}

      - name: Run benchmarks and compare
        # --benchmark-fail-on-alert will fail if a regression is detected
        # The expression checks if we are in a PR to enable comparison
        run: |
          pytest --benchmark-save=current_run \
                 ${{ github.event_name == 'pull_request' && '--benchmark-compare=current_run' || '' }} \
                 --benchmark-compare-fail=alert:5%

      - name: Upload benchmark data as artifact
        # On the main branch, save the results to the cache
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        uses: actions/cache@v3
        with:
          path: .benchmarks
          key: benchmark-cache-${{ github.ref_name }}
```

**Key parts of this workflow:**
-   **`actions/cache`**: This action is used to store the `.benchmarks` directory. On a PR, it downloads the cache from the `main` branch. On a push to `main`, it saves the new results to the cache.
-   **`--benchmark-compare-fail=alert:5%`**: This is the magic flag. It tells `pytest-benchmark` to compare the current run to the default comparison data (which we loaded from the cache). If any benchmark is more than 5% slower (`alert:5%`), it will cause the pytest command to exit with an error code, failing the CI job.

### Important Considerations for CI Benchmarking

-   **Noisy Neighbors:** CI runners are often virtual machines sharing hardware with other jobs. This can cause performance results to fluctuate. Don't set your failure threshold too low (e.g., 1%). A 5-10% threshold is often more practical to avoid flaky failures.
-   **Dedicated Runners:** For projects where performance is absolutely paramount, consider running benchmarks on dedicated, self-hosted hardware. This provides a much more stable environment for consistent measurements.
-   **Treat Regressions as Data:** A failed performance test in CI isn't a failure; it's a conversation starter. It forces the developer and reviewer to ask: "Is this slowdown expected and acceptable for the new feature, or is it an accidental regression that needs to be fixed?" This makes performance a conscious part of the development process.
