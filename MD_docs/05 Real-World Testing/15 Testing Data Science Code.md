# Chapter 15: Testing Data Science Code

## Challenges in Data Science Testing

## Challenges in Data Science Testing

Testing data science and machine learning code presents a unique set of challenges that differ significantly from traditional software engineering. While the core principles of testing remain the same—verifying correctness and preventing regressions—the nature of the code and its outputs requires specialized tools and techniques.

Understanding these challenges is the first step toward writing effective tests for your data-driven applications.

### Key Challenges

1.  **Floating-Point Precision:**
    Machine learning algorithms are built on linear algebra and calculus, which means they operate heavily on floating-point numbers. Comparing two floats for exact equality (`a == b`) is notoriously unreliable due to the way computers represent these numbers. A tiny, insignificant difference in the 15th decimal place can cause a test to fail, even if the result is functionally correct.

2.  **Non-Determinism and Stochasticity:**
    Many algorithms, from random forests to neural network weight initialization, have a random component. Running the same training code twice might produce two slightly different models with slightly different predictions. Tests that expect exact output values will be flaky and unreliable.

3.  **Large and Complex Data:**
    Data science code operates on data, often large datasets. Unit tests should be fast and isolated, which means you can't run them on your entire 10 GB dataset. Creating small, representative, and manageable sample datasets for testing is a significant challenge in itself.

4.  **"Correctness" is Fuzzy:**
    What does it mean for a machine learning model to be "correct"? Unlike a function that sorts a list, a model's output isn't binary right or wrong. Its performance is statistical. We can't assert that `model.predict(input) == 42.5`. Instead, we need to test for properties: Are the predictions within a plausible range? Is the model's performance on a known subset of data above a certain threshold?

5.  **Long-Running Processes:**
    Training a model can take hours or even days. You cannot include a full training run in your CI/CD pipeline's test suite. Testing strategies must be adapted to verify the training *logic* without executing the entire expensive process.

6.  **Coupling of Code and Data:**
    The behavior of your code is inextricably linked to the data it processes. A data cleaning function might work perfectly until it encounters a new, unexpected category or data format. Your tests need to be robust to variations in data, and you need a way to manage the test data itself.

In this chapter, we will tackle these challenges head-on. We will build a small data processing and modeling pipeline and, step-by-step, introduce pytest techniques to test it robustly and effectively.

## Testing Data Processing Functions

## Testing Data Processing Functions

The foundation of any data science project is data preparation. Functions that clean, transform, and engineer features are the most testable part of the pipeline because their behavior is often deterministic. If you provide the same input, you should get the same output.

This makes them the perfect place to start building our testing skills.

### Phase 1: Establish the Reference Implementation

Let's establish our **anchor example**: a function that preprocesses a dataset of house listings. The function will take a pandas DataFrame, fill missing values in the `year_built` column with the median year, and one-hot encode the `style` column (e.g., 'Ranch', 'Colonial').

Here is the initial implementation.

**The code to be tested:** `data_processing.py`

```python
# data_processing.py
import pandas as pd

def preprocess_data(df: pd.DataFrame) -> pd.DataFrame:
    """
    Preprocesses the housing data.

    - Fills missing 'year_built' with the median.
    - One-hot encodes the 'style' column.
    """
    processed_df = df.copy()

    # Fill missing year_built with the median
    if 'year_built' in processed_df.columns:
        median_year = processed_df['year_built'].median()
        processed_df['year_built'].fillna(median_year, inplace=True)

    # One-hot encode style
    if 'style' in processed_df.columns:
        style_dummies = pd.get_dummies(processed_df['style'], prefix='style', dtype=float)
        processed_df = pd.concat([processed_df, style_dummies], axis=1)
        processed_df.drop('style', axis=1, inplace=True)

    return processed_df
```

### Iteration 0: The First Test

Our first test will verify the basic functionality. We'll create a small, representative DataFrame in the test itself, define the exact expected output, and use pandas' built-in `testing.assert_frame_equal` to compare them.

**The test file:** `test_data_processing.py`

```python
# test_data_processing.py
import pandas as pd
import numpy as np
from pandas.testing import assert_frame_equal
from data_processing import preprocess_data

def test_preprocess_data_basic():
    # 1. Setup: Create the input data
    raw_data = {
        'price': [200000, 350000, 275000, 500000],
        'bedrooms': [3, 4, 3, 5],
        'year_built': [1985.0, np.nan, 1999.0, 2010.0],
        'style': ['Ranch', 'Colonial', 'Ranch', 'Modern']
    }
    input_df = pd.DataFrame(raw_data)

    # 2. Action: Call the function under test
    actual_df = preprocess_data(input_df)

    # 3. Assertion: Define the expected outcome and compare
    # The median of [1985, 1999, 2010] is 1999.0
    expected_data = {
        'price': [200000, 350000, 275000, 500000],
        'bedrooms': [3, 4, 3, 5],
        'year_built': [1985.0, 1999.0, 1999.0, 2010.0],
        'style_Colonial': [0.0, 1.0, 0.0, 0.0],
        'style_Modern': [0.0, 0.0, 0.0, 1.0],
        'style_Ranch': [1.0, 0.0, 1.0, 0.0],
    }
    expected_df = pd.DataFrame(expected_data)

    # Pandas provides a testing utility to compare DataFrames
    assert_frame_equal(actual_df, expected_df)
```

Let's run this test.

```bash
$ pytest
============================= test session starts ==============================
...
collected 1 item

test_data_processing.py .                                                [100%]

============================== 1 passed in ...s ===============================
```

It passes. This test correctly verifies our initial requirements. It's clear, self-contained, and deterministic.

However, data processing pipelines rarely stop here. They evolve. What happens when we add a step that introduces floating-point numbers, like feature scaling? This is where our simple, exact-match approach will break down.

## Approximate Assertions with pytest-approx

## Approximate Assertions with pytest-approx

Our current test works because all our outputs are integers or clean floats. The real world of data science is messy and filled with imprecise floating-point numbers. Let's see what happens when our preprocessing function evolves to handle this reality.

### Iteration 1: Introducing Floating-Point Scaling

**Current state recap:** Our test verifies filling missing values and one-hot encoding.

**Current limitation:** The test relies on exact equality, which is fragile for numerical computations.

**New scenario introduction:** Let's add a feature scaling step to our `preprocess_data` function. We'll scale the `price` column to be between 0 and 1 (a common technique called Min-Max Scaling).

Here's the updated `data_processing.py`.

```python
# data_processing.py (UPDATED)
import pandas as pd

def preprocess_data(df: pd.DataFrame) -> pd.DataFrame:
    """
    Preprocesses the housing data.

    - Fills missing 'year_built' with the median.
    - One-hot encodes the 'style' column.
    - Min-Max scales the 'price' column.
    """
    processed_df = df.copy()

    # Fill missing year_built with the median
    if 'year_built' in processed_df.columns:
        median_year = processed_df['year_built'].median()
        processed_df['year_built'].fillna(median_year, inplace=True)

    # One-hot encode style
    if 'style' in processed_df.columns:
        style_dummies = pd.get_dummies(processed_df['style'], prefix='style', dtype=float)
        processed_df = pd.concat([processed_df, style_dummies], axis=1)
        processed_df.drop('style', axis=1, inplace=True)

    # --- NEW ---
    # Min-Max scale the price
    if 'price' in processed_df.columns:
        min_price = processed_df['price'].min()
        max_price = processed_df['price'].max()
        processed_df['price'] = (processed_df['price'] - min_price) / (max_price - min_price)
    # --- END NEW ---

    return processed_df
```

Now, we must update our test to reflect this new expected output. The prices are no longer `200000`, `350000`, etc., but scaled values between 0 and 1.

**Updated test file:** `test_data_processing.py`

```python
# test_data_processing.py (UPDATED)
import pandas as pd
import numpy as np
from pandas.testing import assert_frame_equal
from data_processing import preprocess_data

def test_preprocess_data_with_scaling():
    # 1. Setup
    raw_data = {
        'price': [200000, 350000, 275000, 500000],
        'bedrooms': [3, 4, 3, 5],
        'year_built': [1985.0, np.nan, 1999.0, 2010.0],
        'style': ['Ranch', 'Colonial', 'Ranch', 'Modern']
    }
    input_df = pd.DataFrame(raw_data)

    # 2. Action
    actual_df = preprocess_data(input_df)

    # 3. Assertion
    # min=200k, max=500k.
    # 200k -> (200-200)/(500-200) = 0.0
    # 350k -> (350-200)/(500-200) = 150/300 = 0.5
    # 275k -> (275-200)/(500-200) = 75/300 = 0.25
    # 500k -> (500-200)/(500-200) = 1.0
    expected_data = {
        'price': [0.0, 0.5, 0.25, 1.0], # <-- Updated values
        'bedrooms': [3, 4, 3, 5],
        'year_built': [1985.0, 1999.0, 1999.0, 2010.0],
        'style_Colonial': [0.0, 1.0, 0.0, 0.0],
        'style_Modern': [0.0, 0.0, 0.0, 1.0],
        'style_Ranch': [1.0, 0.0, 1.0, 0.0],
    }
    expected_df = pd.DataFrame(expected_data)

    # This will fail due to floating point issues!
    assert_frame_equal(actual_df, expected_df)
```

### Failure Demonstration

Let's run this updated test. Even though our manual calculation seems correct, we get a failure.

```bash
$ pytest
============================= test session starts ==============================
...
collected 1 item

test_data_processing.py F                                                [100%]

=================================== FAILURES ===================================
______________________ test_preprocess_data_with_scaling _______________________

    def test_preprocess_data_with_scaling():
...
        # This will fail due to floating point issues!
>       assert_frame_equal(actual_df, expected_df)
E       AssertionError: DataFrame.iloc[:, 0] (column name="price") are different
E
E       DataFrame.iloc[:, 0] (column name="price") values are different (25.0 %)
E       [index]: [0, 1, 2, 3]
E       [left]:  [0.0, 0.5, 0.25, 1.0]
E       [right]: [0.0, 0.5, 0.25, 1.0]

test_data_processing.py:35: AssertionError
=========================== short test summary info ============================
FAILED test_data_processing.py::test_preprocess_data_with_scaling - AssertionError...
============================== 1 failed in ...s ================================
```

### Diagnostic Analysis: Reading the Failure

This failure is subtle and classic in data science testing.

**The complete output**:
```
FAILED test_data_processing.py::test_preprocess_data_with_scaling - AssertionError: DataFrame.iloc[:, 0] (column name="price") are different

DataFrame.iloc[:, 0] (column name="price") values are different (25.0 %)
[index]: [0, 1, 2, 3]
[left]:  [0.0, 0.5, 0.25, 1.0]
[right]: [0.0, 0.5, 0.25, 1.0]
```

**Let's parse this section by section**:

1.  **The summary line**: `AssertionError: DataFrame.iloc[:, 0] (column name="price") are different`
    -   What this tells us: The pandas `assert_frame_equal` function found a discrepancy. It's helpfully telling us the problem is in the first column, named "price".

2.  **The assertion introspection**:
    ```
    [left]:  [0.0, 0.5, 0.25, 1.0]
    [right]: [0.0, 0.5, 0.25, 1.0]
    ```
    -   What this tells us: This is the confusing part. The values *look identical*! This is the classic signature of a floating-point precision issue. The actual difference is likely at a very small decimal place that isn't being printed in the summary. The underlying values might be `0.5` and `0.5000000000000001`.

**Root cause identified**: We are using an exact equality check (`assert_frame_equal`) on columns containing floating-point numbers, which is unreliable.
**Why the current approach can't solve this**: `assert_frame_equal` by default demands bit-for-bit equality. It's too strict for numerical work.
**What we need**: A way to assert that two numbers are "close enough" or approximately equal.

### Technique Introduced: `pytest.approx`

Pytest provides a powerful and intuitive tool for this exact problem: `pytest.approx`. It allows you to check if a number is equal to another number within a certain tolerance.

You can use it like this:
`assert actual_float == pytest.approx(expected_float)`

It works with simple numbers, lists, NumPy arrays, and dictionaries of numbers.

### Solution Implementation

Instead of comparing the entire DataFrame at once, we can separate our checks. We'll check the non-numeric columns for exact equality and the numeric `price` column for approximate equality.

**Before (in `test_data_processing.py`):**

```python
# ...
    # This will fail due to floating point issues!
    assert_frame_equal(actual_df, expected_df)
```

**After (in `test_data_processing.py`):**

```python
# test_data_processing.py (FIXED)
import pandas as pd
import numpy as np
from pandas.testing import assert_frame_equal
from pytest import approx  # <-- Import approx
from data_processing import preprocess_data

def test_preprocess_data_with_scaling_and_approx():
    # 1. Setup
    raw_data = {
        'price': [200000, 350000, 275000, 500000],
        'bedrooms': [3, 4, 3, 5],
        'year_built': [1985.0, np.nan, 1999.0, 2010.0],
        'style': ['Ranch', 'Colonial', 'Ranch', 'Modern']
    }
    input_df = pd.DataFrame(raw_data)

    # 2. Action
    actual_df = preprocess_data(input_df)

    # 3. Assertion
    expected_prices = [0.0, 0.5, 0.25, 1.0]
    expected_non_numeric_data = {
        'bedrooms': [3, 4, 3, 5],
        'year_built': [1985.0, 1999.0, 1999.0, 2010.0],
        'style_Colonial': [0.0, 1.0, 0.0, 0.0],
        'style_Modern': [0.0, 0.0, 0.0, 1.0],
        'style_Ranch': [1.0, 0.0, 1.0, 0.0],
    }
    expected_non_numeric_df = pd.DataFrame(expected_non_numeric_data)

    # Assert the numeric column is "close enough"
    assert list(actual_df['price']) == approx(expected_prices)

    # Assert the rest of the columns are exactly equal
    # We need to re-order columns to match for a reliable comparison
    actual_other_cols = actual_df[expected_non_numeric_df.columns]
    assert_frame_equal(actual_other_cols, expected_non_numeric_df)
```

### Verification

Let's run our new test.

```bash
$ pytest
============================= test session starts ==============================
...
collected 1 item

test_data_processing.py .                                                [100%]

============================== 1 passed in ...s ===============================
```

Success! By separating the exact checks from the approximate checks, we've created a robust test that correctly verifies our logic without being sensitive to tiny floating-point inaccuracies.

**Limitation preview:** This test is getting more complex. We're manually defining a lot of data inside the test function. As our pipeline grows, this will become unmanageable. We need a better way to handle test data and setup.

## Testing Machine Learning Models

## Testing Machine Learning Models

We've successfully tested our data processing function. Now we move to the next stage: training and testing a machine learning model. This is where the idea of "correctness" becomes fuzzy. We can't test for exact prediction values, but we can test the model's behavior and properties.

### Iteration 2: Introducing a Model

**Current state recap:** Our test for `preprocess_data` is robust against floating-point issues using `pytest.approx`.

**Current limitation:** We are only testing the data transformation, not the model that will use the data.

**New scenario introduction:** Let's create a simple `train_and_predict` function. It will take raw data, use our `preprocess_data` function, train a basic `scikit-learn` Linear Regression model, and make predictions.

**New file:** `model.py`

```python
# model.py
import pandas as pd
from sklearn.linear_model import LinearRegression
from data_processing import preprocess_data

def train_and_predict(train_df: pd.DataFrame, predict_df: pd.DataFrame) -> list:
    """
    Trains a model on train_df and makes predictions on predict_df.
    """
    # Preprocess both training and prediction data
    processed_train_df = preprocess_data(train_df)
    processed_predict_df = preprocess_data(predict_df)

    # Ensure prediction data has all columns from training, fill missing with 0
    # This handles cases where a 'style' in predict_df was not in train_df
    train_cols = processed_train_df.columns.drop('price')
    processed_predict_df = processed_predict_df.reindex(
        columns=train_cols, fill_value=0
    )

    # Define features (X) and target (y)
    X_train = processed_train_df[train_cols]
    y_train = processed_train_df['price']

    # Train the model
    model = LinearRegression()
    model.fit(X_train, y_train)

    # Make predictions
    X_predict = processed_predict_df[train_cols]
    predictions = model.predict(X_predict)

    return predictions.tolist()
```

### How to Test a Model?

We can't assert `prediction == 12345.67`. So what *can* we test?

1.  **Output Shape:** If we ask for predictions on 3 houses, we should get back 3 predictions.
2.  **Output Type:** The predictions should be a list of floats.
3.  **Prediction Range:** For house prices, predictions should probably be positive. A negative price is a sign something is very wrong.
4.  **Determinism (Sanity Check):** If we train the same model on the same data twice, we should get the same predictions. (Note: This only works for deterministic models like Linear Regression. For stochastic models, you'd need to set a `random_state`).
5.  **Performance Threshold (Advanced):** On a small, fixed dataset, the model's error (e.g., Mean Squared Error) should be below a certain value. This acts as a regression test for model performance.

Let's write a test that covers the first three points.

**New test file:** `test_model.py`

```python
# test_model.py
import pandas as pd
import numpy as np
from model import train_and_predict

def test_train_and_predict_properties():
    # 1. Setup: Create training data and data for prediction
    train_data = {
        'price': [200000, 350000, 275000, 500000, 450000],
        'bedrooms': [3, 4, 3, 5, 4],
        'year_built': [1985.0, 1992.0, 1999.0, 2010.0, 2005.0],
        'style': ['Ranch', 'Colonial', 'Ranch', 'Modern', 'Colonial']
    }
    train_df = pd.DataFrame(train_data)

    predict_data = {
        'bedrooms': [3, 5],
        'year_built': [1990.0, 2015.0],
        'style': ['Ranch', 'Modern']
    }
    predict_df = pd.DataFrame(predict_data)

    # 2. Action
    predictions = train_and_predict(train_df, predict_df)

    # 3. Assertions
    # Test output shape
    assert len(predictions) == 2

    # Test output type
    assert all(isinstance(p, float) for p in predictions)

    # Test prediction range (a simple heuristic)
    # Given our training prices, predictions shouldn't be negative or astronomical.
    assert all(p > 0 for p in predictions)
```

Let's run our new test suite.

```bash
$ pytest
============================= test session starts ==============================
...
collected 2 items

test_data_processing.py .                                                [ 50%]
test_model.py .                                                          [100%]

============================== 2 passed in ...s ===============================
```

It passes. Our test successfully verifies the *behavior* and *contract* of our prediction function without being coupled to specific prediction values. This is a much more robust way to test machine learning models.

**Limitation preview:** Look at all that setup code! We're creating DataFrames inside `test_data_processing.py` and `test_model.py`. This is repetitive and violates the DRY (Don't Repeat Yourself) principle. If we need to add a new column, we have to update it in multiple places. There must be a better way to manage this shared test data setup.

## Fixtures for Data Pipelines

## Fixtures for Data Pipelines

As we've seen, testing data science code involves a lot of setup. We need to create DataFrames, load data, and sometimes even train a model *before* our test can even begin. Pytest fixtures are the perfect solution for managing this setup code, making our tests cleaner, more modular, and more efficient.

### Iteration 3: Refactoring Setup with Fixtures

**Current state recap:** We have two test files, both with significant, manually-created pandas DataFrames inside the test functions.

**Current limitation:** The setup code is duplicated and tightly coupled to the tests. This makes maintenance difficult and tests harder to read.

**Technique introduced:** We will use pytest fixtures to externalize the creation of our test data. We'll create a `conftest.py` file to hold fixtures that can be shared across our entire test suite.

We will create two fixtures:
1.  `raw_housing_data()`: Provides a raw, unprocessed DataFrame for testing.
2.  `processed_housing_data()`: Depends on the first fixture, runs the data through `preprocess_data`, and provides the processed DataFrame.

**New file:** `conftest.py`

```python
# conftest.py
import pytest
import pandas as pd
import numpy as np
from data_processing import preprocess_data

@pytest.fixture(scope="module")
def raw_housing_data() -> pd.DataFrame:
    """Fixture for a raw housing data DataFrame."""
    raw_data = {
        'price': [200000, 350000, 275000, 500000, 450000],
        'bedrooms': [3, 4, 3, 5, 4],
        'year_built': [1985.0, np.nan, 1999.0, 2010.0, 2005.0],
        'style': ['Ranch', 'Colonial', 'Ranch', 'Modern', 'Colonial']
    }
    return pd.DataFrame(raw_data)

@pytest.fixture(scope="module")
def processed_housing_data(raw_housing_data: pd.DataFrame) -> pd.DataFrame:
    """Fixture for processed housing data."""
    return preprocess_data(raw_housing_data)
```

By placing these in `conftest.py`, they are automatically available to all tests. The `scope="module"` means the fixture will be set up only once per test module, which is more efficient than setting it up for every single test function.

### Solution Implementation

Now we can refactor our test files to use these fixtures. The tests become dramatically simpler and more focused on their specific assertion logic.

**Before (in `test_data_processing.py`):**

```python
# test_data_processing.py (OLD version)
def test_preprocess_data_with_scaling_and_approx():
    # 1. Setup
    raw_data = {
        'price': [200000, 350000, 275000, 500000],
        'bedrooms': [3, 4, 3, 5],
        'year_built': [1985.0, np.nan, 1999.0, 2010.0],
        'style': ['Ranch', 'Colonial', 'Ranch', 'Modern']
    }
    input_df = pd.DataFrame(raw_data)

    # 2. Action
    actual_df = preprocess_data(input_df)
    # ... assertions ...
```

**After (in `test_data_processing.py`):**

```python
# test_data_processing.py (REFACTORED)
import pandas as pd
from pytest import approx
from pandas.testing import assert_frame_equal

def test_preprocess_data_output(processed_housing_data: pd.DataFrame):
    # The 'processed_housing_data' fixture handles all setup and action!
    # We just need to assert on the result.
    actual_df = processed_housing_data

    # Assertions
    # The median of [1985, 1999, 2010, 2005] is 2002.0
    # min_price=200k, max_price=500k
    expected_prices = [0.0, 0.5, 0.25, 1.0, 0.83333333]
    assert list(actual_df['price']) == approx(expected_prices)
    assert actual_df['year_built'].isnull().sum() == 0
    assert actual_df['year_built'][1] == 2002.0 # Check median fill
    assert 'style' not in actual_df.columns
    assert 'style_Ranch' in actual_df.columns
```

And now for `test_model.py`.

**Before (in `test_model.py`):**

```python
# test_model.py (OLD version)
def test_train_and_predict_properties():
    # 1. Setup: Create training data and data for prediction
    train_data = { ... }
    train_df = pd.DataFrame(train_data)
    predict_data = { ... }
    predict_df = pd.DataFrame(predict_data)

    # 2. Action
    predictions = train_and_predict(train_df, predict_df)
    # ... assertions ...
```

**After (in `test_model.py`):**

```python
# test_model.py (REFACTORED)
import pandas as pd
from model import train_and_predict

def test_train_and_predict_properties_fixture(raw_housing_data: pd.DataFrame):
    # 1. Setup: Use the fixture for training data
    train_df = raw_housing_data

    # Create a smaller, specific dataset for prediction
    predict_data = {
        'bedrooms': [3, 5],
        'year_built': [1990.0, 2015.0],
        'style': ['Ranch', 'Modern']
    }
    predict_df = pd.DataFrame(predict_data)

    # 2. Action
    predictions = train_and_predict(train_df, predict_df)

    # 3. Assertions
    assert len(predictions) == 2
    assert all(isinstance(p, float) for p in predictions)
    assert all(p > 0 for p in predictions)
```

### Verification

Let's run the full suite again.

```bash
$ pytest -v
============================= test session starts ==============================
...
collected 2 items

test_data_processing.py::test_preprocess_data_output PASSED              [ 50%]
test_model.py::test_train_and_predict_properties_fixture PASSED          [100%]

============================== 2 passed in ...s ===============================
```

The tests still pass, but now our test code is dramatically cleaner. The logic for creating data is centralized in `conftest.py`. If we need to update the test dataset, we only have to do it in one place. The tests themselves are now focused purely on the behavior they are meant to verify. This is a huge win for maintainability.

**Limitation preview:** Our pipeline produces not just numbers, but also visualizations. How can we test a function that creates a plot? We can't easily compare images in an automated test.

## Testing Visualization Code

## Testing Visualization Code

Testing code that generates plots (e.g., with Matplotlib or Seaborn) is notoriously difficult. You can't easily perform an assertion on a PNG file. A single-pixel difference would cause a test to fail, making such tests extremely brittle.

However, this doesn't mean we can't test visualization code at all. We just need to change our approach. Instead of testing the visual *output*, we test the plot-generating *process* and the *properties* of the resulting plot object.

### Iteration 4: Introducing a Plotting Function

**Current state recap:** Our data processing and model prediction logic is well-tested and uses fixtures for clean setup.

**Current limitation:** We have no tests for any visualization outputs of our project.

**New scenario introduction:** Let's add a function that takes a trained model and feature names, and generates a bar chart of feature importances (or coefficients, in the case of linear regression).

**New file:** `visualization.py`

```python
# visualization.py
import pandas as pd
import matplotlib.pyplot as plt
from sklearn.linear_model import LinearRegression

def plot_feature_coefficients(model: LinearRegression, feature_names: list):
    """
    Generates and shows a bar plot of model feature coefficients.
    """
    if not hasattr(model, 'coef_'):
        raise ValueError("Model does not have coefficients. Was it trained?")

    coefficients = model.coef_
    coef_df = pd.DataFrame({'feature': feature_names, 'coefficient': coefficients})
    coef_df = coef_df.sort_values('coefficient', ascending=False)

    fig, ax = plt.subplots()
    ax.bar(coef_df['feature'], coef_df['coefficient'])
    ax.set_title("Model Feature Coefficients")
    ax.set_xlabel("Feature")
    ax.set_ylabel("Coefficient Value")
    plt.xticks(rotation=45, ha='right')
    plt.tight_layout()
    # In a real script, you might use plt.show() or fig.savefig()
    # For testing, we will want to get the Axes object back.
    return ax
```

Notice the function now returns the `ax` (Matplotlib Axes) object. This is a crucial change that makes the function testable. Functions that only call `plt.show()` are very difficult to test.

### Testing Strategies for Plots

#### Strategy 1: The Smoke Test

The simplest possible test for a plotting function is a "smoke test." It doesn't check for correctness; it only checks if the function runs to completion without raising an error. This is surprisingly useful for catching bugs introduced during refactoring.

#### Strategy 2: Inspecting Plot Properties

A more powerful approach is to inspect the properties of the returned Axes object. We can't see the plot, but we can ask questions about its structure:
-   Does it have the correct title?
-   Are the axis labels set correctly?
-   Does it have the right number of bars?

Let's create a test file that does both. First, we need a fixture to provide a trained model.

**Add to `conftest.py`:**

```python
# conftest.py (add new fixture)
from sklearn.linear_model import LinearRegression
from model import preprocess_data # Assuming preprocess_data is accessible

# ... existing fixtures ...

@pytest.fixture(scope="module")
def trained_linear_model(processed_housing_data: pd.DataFrame):
    """Fixture for a trained LinearRegression model."""
    df = processed_housing_data
    
    X_train = df.drop('price', axis=1)
    y_train = df['price']
    
    model = LinearRegression()
    model.fit(X_train, y_train)
    return model, list(X_train.columns)
```

Now we can write the tests.

**New test file:** `test_visualization.py`

```python
# test_visualization.py
import matplotlib
# Use a non-interactive backend for tests to prevent plots from popping up
matplotlib.use('Agg')
import matplotlib.pyplot as plt
from visualization import plot_feature_coefficients

# Smoke test: Does it run without error?
def test_plot_feature_coefficients_smoke(trained_linear_model):
    model, feature_names = trained_linear_model
    try:
        plot_feature_coefficients(model, feature_names)
        # Close the plot to free up memory
        plt.close()
    except Exception as e:
        assert False, f"Plotting function raised an exception: {e}"

# Property test: Does the plot have the right components?
def test_plot_feature_coefficients_properties(trained_linear_model):
    model, feature_names = trained_linear_model
    ax = plot_feature_coefficients(model, feature_names)

    # Test title
    assert ax.get_title() == "Model Feature Coefficients"

    # Test axis labels
    assert ax.get_xlabel() == "Feature"
    assert ax.get_ylabel() == "Coefficient Value"

    # Test number of bars
    # There should be one bar for each feature
    num_features = len(feature_names)
    assert len(ax.patches) == num_features

    # Close the plot to free up memory
    plt.close()
```

### Verification

Let's run the final test suite.

```bash
$ pytest
============================= test session starts ==============================
...
collected 4 items

test_data_processing.py .                                                [ 25%]
test_model.py .                                                          [ 50%]
test_visualization.py ..                                                 [100%]

============================== 4 passed in ...s ===============================
```

All tests pass. We have successfully tested our visualization code without resorting to fragile image comparison. We've verified that it runs and that the key components of the plot are generated as expected.

### The Journey: From Problem to Solution

| Iteration | Failure Mode / Challenge                               | Technique Applied          | Result                                                              |
| --------- | ------------------------------------------------------ | -------------------------- | ------------------------------------------------------------------- |
| 0         | Need to verify data transformation logic.              | `pandas.testing`           | Basic test for integers and one-hot encoding works.                 |
| 1         | Exact comparisons fail with floating-point numbers.    | `pytest.approx`            | Test is now robust to small floating-point inaccuracies.            |
| 2         | Model predictions are not deterministic or exact.      | Property-based testing     | Tests verify shape, type, and range of predictions, not exact values. |
| 3         | Test setup code is duplicated and hard to maintain.    | `pytest` fixtures          | Setup logic is centralized, making tests clean and maintainable.    |
| 4         | Visual outputs (plots) cannot be easily asserted on.   | Smoke & property testing   | Tests verify the plotting function runs and generates correct components. |

This chapter has demonstrated a robust, layered approach to testing a data science pipeline. By understanding the unique challenges and applying the right tools—`pytest.approx` for floats, property-based assertions for models, and fixtures for setup—you can build a reliable test suite that supports, rather than hinders, your data science workflow.
