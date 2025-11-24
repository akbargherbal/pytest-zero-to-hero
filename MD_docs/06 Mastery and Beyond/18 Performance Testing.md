# Chapter 18: Performance Testing

## When Performance Testing Matters

## Performance: The Invisible Feature

Performance is a critical feature of any application. A function that returns the correct result but takes ten seconds to do so might be functionally correct but practically useless. Similarly, a test suite that takes hours to run becomes a bottleneck for development, slowing down feedback loops and discouraging frequent testing.

In the context of pytest, performance testing splits into two distinct domains:

1.  **Application Performance Testing**: Measuring the speed and resource consumption (CPU, memory) of your actual application code. The goal is to identify bottlenecks, validate optimizations, and prevent performance regressions. For example, "Does this database query run in under 50ms?" or "Does this data processing function leak memory?"

2.  **Test Suite Performance Testing**: Measuring the execution time of your tests themselves. The goal here is to keep your CI/CD pipeline fast and responsive. A slow test suite is a drag on productivity. The focus is on identifying and optimizing slow tests, not necessarily the application code they are testing.

This chapter will equip you with the tools and techniques to tackle both domains. We'll start by benchmarking application code to ensure it's fast and efficient, and then we'll turn our attention to keeping the test suite itself lean and quick.

While pytest is not a dedicated load-testing framework like Locust or JMeter (which are designed to simulate thousands of concurrent users), it is an exceptional tool for **micro-benchmarking**—precisely measuring the performance of individual functions or components under controlled conditions. This is invaluable for catching performance regressions before they ever reach production.

## Measuring Test Execution Time

## The Simplest Tool: `--durations`

Before diving into specialized tools, let's look at a built-in pytest feature that provides a coarse-grained view of test performance: the `--durations` option.

This option reports the `N` slowest test items. It's a fantastic starting point for identifying major bottlenecks in your test suite.

### Phase 1: Establish the Reference Implementation

Let's create a simple function we want to analyze. We'll write two versions: a naive, slow implementation and a more optimized one. Our task is to find common elements between two lists.

This will be our **anchor example** for the chapter. We will use it to demonstrate the limitations of simple timing and the power of statistical benchmarking.

First, let's create our utility module.

```python
# utils/list_operations.py

def find_common_elements_naive(list_a, list_b):
    """
    Finds common elements using nested loops.
    This has a time complexity of O(n*m).
    """
    common = []
    for item_a in list_a:
        for item_b in list_b:
            if item_a == item_b:
                common.append(item_a)
                break  # Move to next item in list_a once a match is found
    return common

def find_common_elements_optimized(list_a, list_b):
    """
    Finds common elements using set intersection.
    This has a time complexity of O(n+m).
    """
    set_a = set(list_a)
    set_b = set(list_b)
    return list(set_a.intersection(set_b))
```

Now, let's write a simple test for the naive version. We'll add a `time.sleep()` to simulate a slow operation and ensure it shows up in our report.

```python
# tests/test_performance.py
import time
import pytest
from utils.list_operations import find_common_elements_naive

@pytest.fixture
def sample_lists():
    """Provides two lists for testing."""
    list_a = list(range(100))
    list_b = list(range(50, 150))
    return list_a, list_b

def test_find_common_elements_correctness(sample_lists):
    """A standard functional test to ensure correctness."""
    list_a, list_b = sample_lists
    result = find_common_elements_naive(list_a, list_b)
    expected = list(range(50, 100))
    assert sorted(result) == expected

def test_slow_operation():
    """A test that is intentionally slow."""
    time.sleep(0.5)
    assert True

def test_fast_operation():
    """A test that is very fast."""
    assert 1 + 1 == 2
```

### Iteration 1: Identifying Slow Tests

Our goal is to find the slowest parts of our test suite. We suspect `test_slow_operation` is a problem, but in a large suite, we wouldn't know where to look.

Let's run pytest with `--durations=3` to see the top 3 slowest items. The duration includes the test function execution time plus its setup and teardown phases.

```bash
pytest --durations=3
```

### Diagnostic Analysis: Reading the Output

**The complete output**:
```bash
============================= test session starts ==============================
...
collected 3 items

tests/test_performance.py ...                                            [100%]

=========================== slowest 3 test durations ===========================
0.50s call     tests/test_performance.py::test_slow_operation
0.00s call     tests/test_performance.py::test_find_common_elements_correctness
0.00s setup    tests/test_performance.py::test_find_common_elements_correctness
0.00s call     tests/test_performance.py::test_fast_operation
0.00s teardown tests/test_performance.py::test_find_common_elements_correctness
0.00s setup    tests/test_performance.py::test_slow_operation
...
============================== 3 passed in 0.51s ===============================
```

**Let's parse this section by section**:

1.  **The summary table**: `slowest 3 test durations`
    -   What this tells us: It lists the slowest items, breaking them down into `setup`, `call` (the test body itself), and `teardown`. This is crucial for diagnosing whether the slowness is in the test logic or the fixture setup.

2.  **The top entry**: `0.50s call tests/test_performance.py::test_slow_operation`
    -   What this tells us: Unsurprisingly, the `call` phase of `test_slow_operation` took about half a second, which directly corresponds to our `time.sleep(0.5)`.

3.  **The other entries**: The other tests and setup phases are extremely fast, registering as `0.00s`.

**Root cause identified**: The `test_slow_operation` is, by far, the slowest test.
**Why the current approach is limited**: While `--durations` is excellent for finding slow *tests*, it's a poor tool for benchmarking *code*. The measurements are noisy, include pytest overhead, and are based on a single run. If we ran this command again, the exact millisecond values would fluctuate. This makes it unreliable for comparing two fast functions.
**What we need**: A tool that can run a piece of code many times in isolation, perform statistical analysis, and give us a reliable, repeatable measurement. This is called micro-benchmarking.

## pytest-benchmark for Reliable Benchmarks

To get reliable performance measurements, we need a specialized tool. The most popular and powerful option in the pytest ecosystem is `pytest-benchmark`.

First, install it:

```bash
pip install pytest-benchmark
```

`pytest-benchmark` provides a `benchmark` fixture that handles the complexity of running your code multiple times, measuring execution time accurately, and calculating statistics like mean, median, standard deviation, and more.

### Iteration 2: Introducing `pytest-benchmark`

Let's try to benchmark our `find_common_elements_naive` function. Using `--durations` was ineffective because the function is too fast to be measured reliably in a single run. `pytest-benchmark` solves this.

Here's how you use the `benchmark` fixture: you call it like a function, passing the callable you want to benchmark as the first argument, followed by its arguments.

**Before: The old functional test**

```python
# tests/test_performance.py (excerpt)

def test_find_common_elements_correctness(sample_lists):
    """A standard functional test to ensure correctness."""
    list_a, list_b = sample_lists
    result = find_common_elements_naive(list_a, list_b)
    expected = list(range(50, 100))
    assert sorted(result) == expected
```

**After: The new benchmark test**

```python
# tests/test_performance.py (add this test)
from utils.list_operations import find_common_elements_naive

def test_find_common_elements_naive_performance(benchmark, sample_lists):
    """Benchmarks the naive implementation."""
    list_a, list_b = sample_lists
    # The benchmark fixture takes the function to run, and its args/kwargs
    result = benchmark(find_common_elements_naive, list_a, list_b)
    
    # You can still add assertions to ensure correctness!
    expected = list(range(50, 100))
    assert sorted(result) == expected
```

Now, let's run pytest. `pytest-benchmark` is automatically active when installed.

```bash
pytest tests/test_performance.py::test_find_common_elements_naive_performance
```

**The complete output**:
```bash
============================= test session starts ==============================
...
plugins: benchmark-4.0.0, ...
...
collected 1 item

tests/test_performance.py::test_find_common_elements_naive_performance 
-------------------------------- benchmark: 1 tests --------------------------------
Name (time in ms)                                        Min      Max     Mean   StdDev  Median     IQR  Outliers     OPS  Rounds  Iterations
--------------------------------------------------------------------------------------------------------------------------------------------
test_find_common_elements_naive_performance         1.0315   1.5329   1.0861   0.0981  1.0524   0.0497     15;10   920.758     172           1
--------------------------------------------------------------------------------------------------------------------------------------------
PASSED                                                                   [100%]

============================== 1 passed in 2.05s ===============================
```

### Diagnostic Analysis: Reading the Benchmark Report

This table is packed with statistical data, which is why it's so much more reliable than a single timing run.

1.  **Name**: The name of the test function.
2.  **Min, Max, Mean, Median**: These are the core statistics of the execution time over many runs (called "rounds"). The `Mean` is the average time, but `Median` is often more useful as it's less sensitive to outliers.
3.  **StdDev**: The standard deviation, which measures how much the timings varied. A low `StdDev` indicates a stable, reliable measurement.
4.  **OPS (Operations Per Second)**: This is a very useful metric (`1 / Mean`), telling you how many times the function can run in one second.
5.  **Rounds**: The number of times the benchmark measurement was taken.
6.  **Iterations**: The number of times the code was run within each round. `pytest-benchmark` automatically adjusts this number to get a reasonable total execution time.

**Root cause identified**: We now have a statistically sound measurement of our function's performance.
**What we need**: A way to compare this naive implementation against our optimized one.

### Iteration 3: Comparing Implementations with Parametrization

`pytest-benchmark` shines when comparing different approaches. We can use `pytest.mark.parametrize` to feed both our naive and optimized functions into the same benchmark test. This ensures they are benchmarked under identical conditions.

**Before: A single benchmark test**

```python
# tests/test_performance.py (excerpt)
from utils.list_operations import find_common_elements_naive

def test_find_common_elements_naive_performance(benchmark, sample_lists):
    """Benchmarks the naive implementation."""
    list_a, list_b = sample_lists
    benchmark(find_common_elements_naive, list_a, list_b)
```

**After: A parametrized comparison test**

```python
# tests/test_performance.py (replace previous test with this)
import pytest
from utils.list_operations import (
    find_common_elements_naive,
    find_common_elements_optimized
)

# A list of functions to test
ALGORITHMS = [find_common_elements_naive, find_common_elements_optimized]

@pytest.mark.parametrize("algorithm", ALGORITHMS)
def test_find_common_elements_performance(benchmark, sample_lists, algorithm):
    """Benchmarks and compares different implementations."""
    list_a, list_b = sample_lists
    
    # The benchmark fixture gets the parametrized algorithm
    result = benchmark(algorithm, list_a, list_b)

    # We can still assert correctness
    expected = list(range(50, 100))
    assert sorted(result) == expected
```

Let's run this parametrized test.

```bash
pytest tests/test_performance.py::test_find_common_elements_performance
```

The output now includes a grouped and sorted comparison table, making the performance difference immediately obvious.

**The complete output**:
```bash
============================= test session starts ==============================
...
collected 2 items

tests/test_performance.py ..                                             [100%]
-------------------------------- benchmark: 2 tests --------------------------------
Name (time in us)                                           Min      Max     Mean   StdDev   Median      IQR  Outliers      OPS  Rounds  Iterations
-------------------------------------------------------------------------------------------------------------------------------------------------
test_find_common_elements_performance[optimized]         6.8460   9.9410   7.1359   0.4904   7.0090   0.2295   453;136  140,139.7525    1000           1
test_find_common_elements_performance[naive]         1,029.1000 1,170.2000 1,048.9839 25.1278 1,041.8000  20.4250     23;5    953.3031     176           1
-------------------------------------------------------------------------------------------------------------------------------------------------

============================== 2 passed in 3.12s ===============================
```
The table is automatically sorted from fastest to slowest. The results are stark: the optimized version runs in about 7 **microseconds** (`us`), while the naive version takes over 1000 microseconds (1 **millisecond**). The optimized version is over 100 times faster! This is the kind of insight that simple timing with `--durations` could never provide.

## Memory Profiling in Tests

## Memory: The Other Side of Performance

Performance isn't just about speed; it's also about memory consumption. A function that's lightning-fast but consumes gigabytes of RAM can be just as problematic as a slow one. To profile memory usage, we can use the `pytest-memray` plugin, which integrates the powerful `memray` memory profiler into pytest.

First, install the necessary packages:

```bash
pip install pytest-memray memray
```

### Phase 1: Establish the Reference Implementation

Let's create a function that is deliberately memory-intensive. It will build a large data structure in memory.

```python
# utils/memory_operations.py

def create_large_object(size):
    """
    Creates a list containing many dictionaries.
    This is designed to consume a significant amount of memory.
    """
    # Each dictionary is a small object, but we create millions of them.
    return [{"id": i, "data": "x" * 10} for i in range(size)]
```

And a simple test for it:

```python
# tests/test_memory.py
from utils.memory_operations import create_large_object

def test_create_large_object():
    """Tests the memory-intensive function."""
    # Create 1 million objects
    data = create_large_object(1_000_000)
    assert len(data) == 1_000_000
    assert data[999]["id"] == 999
```

This test passes, but it tells us nothing about how much memory was used.

### Iteration 1: Gaining Visibility with `--memray`

The "failure" here is a lack of visibility. We have no idea if our function is efficient or a memory hog. Let's run pytest with the `--memray` flag to enable memory profiling.

```bash
pytest --memray tests/test_memory.py
```

This command runs the tests as usual, but `memray` tracks every memory allocation in the background. After the test run, it generates a report.

**The complete output**:
```bash
============================= test session starts ==============================
...
plugins: memray-1.5.0, ...
...
collected 1 item

tests/test_memory.py .                                                   [100%]
=============================== slowest 1 test durations ===============================
0.34s call     tests/test_memory.py::test_create_large_object
(1 durations hidden)
============================== 1 passed in 0.48s ===============================
Wrote memray report to: memray-tests.test_memory.py::test_create_large_object.bin
You can now generate a report from the stored allocation records.
Some available reporters are:
- memray flamegraph memray-tests.test_memory.py::test_create_large_object.bin
- memray table memray-tests.test_memory.py::test_create_large_object.bin
- memray summary memray-tests.test_memory.py::test_create_large_object.bin
```
The key line is `Wrote memray report to: ...`. `memray` has saved a detailed snapshot of all memory allocations to a `.bin` file. We can now analyze this file. A flame graph is often the most intuitive visualization.

```bash
memray flamegraph memray-tests.test_memory.py::test_create_large_object.bin
```

This command will generate an HTML file and open it in your browser. The flame graph is a powerful visualization where:
-   The width of a bar represents the percentage of memory allocated by that function and its children.
-   The y-axis represents the call stack.

You would see a large bar corresponding to the list comprehension inside `create_large_object`, confirming that it is the source of the vast majority of memory allocations.

### Iteration 2: Enforcing Memory Limits

Visibility is good, but automated prevention of regressions is better. `pytest-memray` allows you to set memory limits for your tests. If a test exceeds this limit, it fails. This is perfect for CI.

We can use the `@pytest.mark.limit_memory` marker. Let's set a limit of 100 MiB and see our test fail.

**Before: No memory limit**

```python
# tests/test_memory.py (excerpt)
from utils.memory_operations import create_large_object

def test_create_large_object():
    data = create_large_object(1_000_000)
    assert len(data) == 1_000_000
```

**After: With a memory limit marker**

```python
# tests/test_memory.py (modified)
import pytest
from utils.memory_operations import create_large_object

@pytest.mark.limit_memory("100 MB")
def test_create_large_object_with_limit():
    """Tests the memory-intensive function with a limit."""
    data = create_large_object(1_000_000)
    assert len(data) == 1_000_000
    assert data[999]["id"] == 999
```

Now, run the test again. It doesn't need the `--memray` flag; the marker is sufficient.

```bash
pytest tests/test_memory.py
```

### Diagnostic Analysis: Reading the Failure

**The complete output**:
```bash
============================= test session starts ==============================
...
collected 1 item

tests/test_memory.py F                                                   [100%]

=================================== FAILURES ===================================
___________________ test_create_large_object_with_limit ____________________

    @pytest.mark.limit_memory("100 MB")
    def test_create_large_object_with_limit():
        """Tests the memory-intensive function with a limit."""
>       data = create_large_object(1_000_000)

tests/test_memory.py:6: 
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 
utils/memory_operations.py:7: in create_large_object
    return [{"id": i, "data": "x" * 10} for i in range(size)]
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ 

HighWatermarkExceededError: The test allocated 123.9MiB, which is more than the 100.0MiB limit.
=========================== short test summary info ============================
FAILED tests/test_memory.py::test_create_large_object_with_limit - HighWater...
============================== 1 failed in 0.45s ===============================
```

**Let's parse this section by section**:

1.  **The summary line**: `FAILED ... - HighWatermarkExceededError`
    -   What this tells us: The test failed not because of an `AssertionError`, but because of a specific error from the `memray` plugin.

2.  **The error message**: `HighWatermarkExceededError: The test allocated 123.9MiB, which is more than the 100.0MiB limit.`
    -   What this tells us: This is an incredibly clear and actionable error. It tells us exactly how much memory was allocated and how much it exceeded the limit by.

**Root cause identified**: The `create_large_object` function allocates more than 100 MiB of memory.
**What we need**: We can now make an informed decision. Either the memory usage is acceptable and we should raise the limit in the test (e.g., `@pytest.mark.limit_memory("150 MB")`), or the function needs to be optimized to use less memory. This marker turns an invisible problem into a concrete, failing test.

## Identifying and Fixing Slow Tests

## Keeping the Test Suite Fast

So far, we've focused on the performance of the application code. Now, let's turn our attention to the test suite itself. As a project grows, the test suite can become slow, increasing CI/CD times and harming developer productivity.

We'll revisit the `--durations` flag, which is the perfect tool for this job.

### The Problem: A Slow Fixture

A common cause of slow test suites is an expensive fixture that is set up more often than necessary. Imagine a fixture that reads a large configuration file or connects to a database. If it's `function`-scoped, this expensive operation will happen before every single test that uses it.

Let's simulate this. We'll create a fixture that simulates reading a large data file by sleeping for a short duration.

```python
# tests/test_slow_suite.py
import pytest
import time

@pytest.fixture
def large_dataset():
    """
    A function-scoped fixture that simulates a slow data load.
    This will run for EACH test that uses it.
    """
    print("\n(Loading large dataset...)")
    time.sleep(0.2)
    return list(range(100))

def test_data_mean(large_dataset):
    assert sum(large_dataset) / len(large_dataset) == 49.5

def test_data_max(large_dataset):
    assert max(large_dataset) == 99

def test_data_min(large_dataset):
    assert min(large_dataset) == 0
```

We have three tests that all use the `large_dataset` fixture. Since the fixture is function-scoped by default, the `time.sleep(0.2)` will run three times. The total run time should be over 0.6 seconds.

Let's confirm this with `--durations`.

```bash
pytest -v --durations=5 tests/test_slow_suite.py
```

### Diagnostic Analysis: Pinpointing the Slowdown

**The complete output**:
```bash
============================= test session starts ==============================
...
collected 3 items

tests/test_slow_suite.py::test_data_mean 
(Loading large dataset...)
PASSED                         [ 33%]
tests/test_slow_suite.py::test_data_max 
(Loading large dataset...)
PASSED                         [ 66%]
tests/test_slow_suite.py::test_data_min 
(Loading large dataset...)
PASSED                         [100%]

=========================== slowest 5 test durations ===========================
0.20s setup    tests/test_slow_suite.py::test_data_min
0.20s setup    tests/test_slow_suite.py::test_data_max
0.20s setup    tests/test_slow_suite.py::test_data_mean
0.00s call     tests/test_slow_suite.py::test_data_min
0.00s call     tests/test_slow_suite.py::test_data_max
============================== 3 passed in 0.61s ===============================
```

**Let's parse this section by section**:

1.  **The print statements**: We see `(Loading large dataset...)` printed three times, confirming our hypothesis that the fixture runs for each test.
2.  **The durations table**: This is the key insight. The three slowest items are all `setup` phases, each taking around 0.2 seconds. The `call` phases are nearly instantaneous.
3.  **The total time**: The suite took `0.61s`, which is roughly `3 * 0.2s` plus a small overhead.

**Root cause identified**: The slowness is not in the test logic (`call`) but in the fixture setup (`setup`). The `large_dataset` fixture is being wastefully re-created for every test.
**What we need**: A way to run the expensive setup operation only once for all tests that need it. This is a perfect use case for changing the fixture's scope.

### The Solution: Changing Fixture Scope

If the data being loaded is read-only and doesn't change between tests, we can change the fixture's scope from `function` to `module` or `session`. This will cause the fixture to be set up only once per module or once per the entire test session, respectively.

**Before: Default `function` scope**

```python
# tests/test_slow_suite.py (excerpt)
@pytest.fixture
def large_dataset():
    # ...
    time.sleep(0.2)
    return list(range(100))
```

**After: Efficient `module` scope**

```python
# tests/test_slow_suite.py (modified)
import pytest
import time

@pytest.fixture(scope="module")
def large_dataset():
    """
    A module-scoped fixture.
    This will run only ONCE for all tests in this file.
    """
    print("\n(Loading large dataset ONCE...)")
    time.sleep(0.2)
    return list(range(100))

# ... tests remain the same ...
def test_data_mean(large_dataset):
    assert sum(large_dataset) / len(large_dataset) == 49.5

def test_data_max(large_dataset):
    assert max(large_dataset) == 99

def test_data_min(large_dataset):
    assert min(large_dataset) == 0
```

Let's run the tests again and observe the dramatic improvement.

```bash
pytest -v --durations=5 tests/test_slow_suite.py
```

**The verification output**:
```bash
============================= test session starts ==============================
...
collected 3 items

tests/test_slow_suite.py::test_data_mean 
(Loading large dataset ONCE...)
PASSED                         [ 33%]
tests/test_slow_suite.py::test_data_max PASSED                         [ 66%]
tests/test_slow_suite.py::test_data_min PASSED                         [100%]

=========================== slowest 5 test durations ===========================
0.20s setup    tests/test_slow_suite.py::test_data_mean
0.00s call     tests/test_slow_suite.py::test_data_min
0.00s call     tests/test_slow_suite.py::test_data_max
0.00s call     tests/test_slow_suite.py::test_data_mean
0.00s teardown tests/test_slow_suite.py::test_data_mean
============================== 3 passed in 0.21s ===============================
```
The improvement is clear:
-   The print statement `(Loading large dataset ONCE...)` appears only once.
-   The total test time has dropped from `0.61s` to `0.21s`, a 3x speedup.
-   The durations table shows only one slow `setup` phase. Pytest is smart enough to attribute the setup time to the first test that requested the fixture.

This simple change—adding `scope="module"`—is one of the most effective ways to speed up a test suite.

## Performance Testing in CI/CD

## Automating Performance Checks

The true power of these tools is realized when they are integrated into your Continuous Integration / Continuous Deployment (CI/CD) pipeline. This allows you to automatically catch performance regressions before they are merged.

### The Journey: From Problem to Solution

| Iteration | Failure Mode                               | Technique Applied                     | Result                                               |
| --------- | ------------------------------------------ | ------------------------------------- | ---------------------------------------------------- |
| 0         | Slow tests are hidden in the suite         | None                                  | Slow CI builds, developer frustration                |
| 1         | Noisy, unreliable performance measurements | `pytest --durations`                  | Can find slow tests, but useless for benchmarking    |
| 2         | Inability to compare code performance      | `pytest-benchmark` fixture            | Reliable, statistical measurement of a single function |
| 3         | No context for performance numbers         | Parametrized benchmark tests          | Direct, side-by-side comparison of implementations   |
| 4         | Memory usage is completely invisible       | `pytest --memray`                     | Detailed reports of memory allocation                |
| 5         | Memory regressions are not prevented       | `@pytest.mark.limit_memory`           | Automated failure if memory usage exceeds a threshold|
| 6         | Inefficient fixtures slow down the suite   | `scope="module"`/`"session"`          | Drastically reduced test suite execution time        |
| 7         | Regressions are caught manually, if at all | CI/CD integration                     | Automated, preventative performance testing          |

### A CI/CD Workflow for Performance

Here is a practical workflow for integrating `pytest-benchmark` into a CI pipeline (e.g., GitHub Actions, GitLab CI).

**Step 1: Establish a Baseline**

On your main branch, run your benchmarks and save the results to a file. This file represents the "known good" performance of your code.

```bash
# This command runs the benchmarks and saves the results to .benchmarks/
pytest --benchmark-save=baseline
```

This will create a JSON file in a `.benchmarks/` directory. Commit this file to your repository. This baseline should be updated periodically as your application's performance characteristics intentionally change.

**Step 2: Compare on Pull Requests**

In your CI configuration for pull requests, add a step that runs the benchmarks and compares them against the saved baseline.

```bash
# This command compares the current run against the saved baseline
# It will fail the build if a statistically significant regression is detected.
pytest --benchmark-compare=baseline --benchmark-compare-fail=mean:5%
```

Let's break down that command:
-   `--benchmark-compare=baseline`: Specifies the group of saved results to compare against.
-   `--benchmark-compare-fail=mean:5%`: This is the crucial part. It tells `pytest-benchmark` to fail the CI job if the `mean` time of any benchmark increases by more than `5%`. You can set thresholds for `min`, `max`, `median`, etc., and use different percentages.

If a developer pushes a change that makes `find_common_elements_optimized` 10% slower, the CI build will fail with a clear message indicating a performance regression.

**Step 3: Monitor Memory and Slow Tests**

While `pytest-benchmark` is great for micro-benchmarks, you should also monitor the overall health of your suite.

-   **Memory Limits**: Keep tests with `@pytest.mark.limit_memory` in your suite. They will automatically fail in CI if a change causes memory usage to spike.
-   **Slow Test Report**: Periodically run `pytest --durations=20` in a CI job and publish the report. This doesn't need to fail the build, but it creates visibility into the test suite's health, allowing the team to proactively refactor the slowest tests.

By combining these techniques, you create a robust, automated safety net that guards not just the correctness of your code, but also its performance and efficiency.
