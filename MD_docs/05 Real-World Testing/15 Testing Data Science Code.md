# Chapter 15: Testing Data Science Code

## Challenges in Data Science Testing

## Challenges in Data Science Testing

Testing standard software is often about deterministic logic: if you provide input A, you must always get output B. A user's password hash must be exact. A sorted list must be perfectly ordered.

Data science code operates in a world of statistics, probabilities, and approximations. This introduces a unique set of challenges that require a shift in our testing mindset. Before we write any code, let's understand the landscape we're navigating.

### 1. Floating-Point Precision
Computers cannot represent all decimal numbers perfectly. This tiny imprecision means that calculations involving floating-point numbers (like `3.14159`) can lead to unexpected results. The classic example is `0.1 + 0.2`, which does not exactly equal `0.3` in most programming languages. For data science, where almost every calculation involves floats, direct equality checks (`==`) are a recipe for failing tests.

### 2. Large and Complex Data
Machine learning models are trained on datasets that can be gigabytes or even terabytes in size. You cannot and should not load your entire production dataset into a unit test. Our tests must be fast and self-contained, which means we need strategies for working with small, representative samples of data that capture the essential characteristics (and edge cases) of the real thing.

### 3. Stochasticity (Randomness)
Many algorithms in data science are stochastic, meaning they involve a random element. Examples include:
-   Randomly splitting data into training and testing sets.
-   Initializing model weights randomly.
-   Algorithms like Stochastic Gradient Descent.

If you train the same model on the same data twice, you might get two slightly different models. A test that asserts a specific prediction value will be "flaky"—sometimes passing, sometimes failing. We need to test the *behavior* of the model, not a specific random outcome.

### 4. Long-Running Processes
Training a deep learning model can take hours, days, or even weeks. A core principle of a good test suite is that it runs quickly, providing feedback in seconds or minutes. We cannot include model training in our standard test suite. Instead, we often test with pre-trained models or models trained on trivial datasets that finish in milliseconds.

### 5. "Correctness" is Statistical, Not Absolute
How do you know if a prediction model is "correct"? Its performance is measured with statistical metrics like accuracy, precision, or Mean Squared Error. A model with 95% accuracy is considered "good," but it's still "wrong" 5% of the time. We cannot write a simple `assert model.predict(input) == expected_output` for every case. Our tests must focus on properties and contracts, not on getting every single prediction right.

### 6. The Environment and Dependencies
Data science projects often have a fragile web of dependencies: specific versions of NumPy, Pandas, Scikit-learn, PyTorch, or TensorFlow. A minor version change in one library can subtly change numerical results, causing tests to fail. Managing this environment is a critical part of ensuring tests are reliable and reproducible.

These challenges might seem daunting, but they are all solvable. The key is to adapt our testing strategies from asserting exact outcomes to verifying stable behaviors, contracts, and properties. In the following sections, we'll tackle each of these challenges with practical patterns and tools.

## Testing Data Processing Functions

## Testing Data Processing Functions

The foundation of any data science pipeline is data processing: cleaning, transforming, and feature engineering. These functions are often pure and deterministic, making them perfect candidates for traditional unit testing. If you can ensure your data processing code is robust, you can have much higher confidence in the models that consume that data.

Let's imagine we have a simple function to process a dataset of house listings. It should convert a price column from a string to a number and create a new feature for the age of the house.

Here's our data processing module.

```python
# data_processing.py
import pandas as pd
from datetime import datetime

def preprocess_housing_data(df: pd.DataFrame) -> pd.DataFrame:
    """
    Cleans and prepares housing data for modeling.

    - Converts 'price' from string (e.g., "$500,000") to float.
    - Calculates 'house_age' from 'year_built'.
    - Drops rows with missing values.
    """
    processed_df = df.copy()

    # 1. Convert price to numeric
    if 'price' in processed_df.columns:
        processed_df['price'] = (
            processed_df['price']
            .replace({'\$': '', ',': ''}, regex=True)
            .astype(float)
        )

    # 2. Calculate house age
    if 'year_built' in processed_df.columns:
        current_year = datetime.now().year
        processed_df['house_age'] = current_year - processed_df['year_built']

    # 3. Drop rows with any missing values
    processed_df.dropna(inplace=True)

    return processed_df
```

A common but flawed approach is to simply "test" this in a Jupyter notebook by running it on some data and visually inspecting the output. This is not a real test—it's not automated, not repeatable, and easily forgotten.

Let's write a proper pytest test. The best way to compare pandas DataFrames is with the `pandas.testing.assert_frame_equal` function. It gives detailed output on exactly how two DataFrames differ.

Our test file will look like this:

```python
# tests/test_data_processing.py
import pandas as pd
import pytest
from datetime import datetime

from data_processing import preprocess_housing_data

def test_preprocess_housing_data():
    # 1. Setup: Create the input data
    raw_data = {
        'price': ['$750,000', '$1,200,000', '$450,000', None],
        'year_built': [2005, 2018, 1990, 2010],
        'sq_ft': [2200, 3100, 1500, 1800]
    }
    input_df = pd.DataFrame(raw_data)

    # 2. Setup: Define the expected output data
    current_year = datetime.now().year
    expected_data = {
        'price': [750000.0, 1200000.0, 450000.0],
        'year_built': [2005, 2018, 1990],
        'sq_ft': [2200, 3100, 1500],
        'house_age': [current_year - 2005, current_year - 2018, current_year - 1990]
    }
    # Note: The row with the missing price is expected to be dropped.
    # We also need to match the index after dropping.
    expected_df = pd.DataFrame(expected_data, index=[0, 1, 2])

    # 3. Execution: Run the function we are testing
    actual_df = preprocess_housing_data(input_df)

    # 4. Assertion: Check if the actual output matches the expected output
    pd.testing.assert_frame_equal(actual_df, expected_df)

def test_preprocess_empty_dataframe():
    # Edge case: what happens with an empty DataFrame?
    input_df = pd.DataFrame({'price': [], 'year_built': []})
    expected_df = pd.DataFrame({'price': [], 'year_built': [], 'house_age': []})

    actual_df = preprocess_housing_data(input_df)

    # The dtypes might differ (object vs float), so check them explicitly
    pd.testing.assert_frame_equal(actual_df, expected_df, check_dtype=False)
```

This approach is powerful because:
1.  **It's explicit:** The `expected_df` clearly documents the function's contract.
2.  **It's automated:** We can run this test anytime we change the `preprocess_housing_data` function to ensure we haven't broken anything.
3.  **It's precise:** `assert_frame_equal` will tell us if a single value, a column name, a data type, or an index is incorrect, which is far more reliable than visual inspection.

Testing your data processing functions is the single most effective way to build a reliable data science pipeline. Get this right, and you've eliminated a massive source of potential bugs.

## Approximate Assertions with pytest-approx

## Approximate Assertions with pytest-approx

As we discussed, floating-point arithmetic is a major challenge in testing numerical code. Let's see this problem in action.

Imagine a simple function that calculates the average rating from a list of scores.

```python
# calculations.py
import numpy as np

def calculate_average_rating(ratings):
    if not ratings:
        return 0.0
    return np.mean(ratings)
```

Now, let's write a test for it. A common scenario might be calculating the average of `[1, 2, 3]`, which is `2.0`. That works fine. But what about a more complex case?

```python
# tests/test_calculations.py
from calculations import calculate_average_rating

def test_average_rating_simple():
    assert calculate_average_rating([1, 2, 3]) == 2.0

def test_average_rating_float_problem():
    # This test will FAIL!
    ratings = [0.1, 0.2, 0.3]
    # The expected result of np.mean([0.1, 0.2, 0.3]) is 0.2
    # But what if we naively expect the average of 0.1 and 0.2 to be 0.15?
    # Let's try a different example where the sum is the issue.
    # The sum of 0.1 + 0.2 is not exactly 0.3
    assert 0.1 + 0.2 == 0.3
```

Let's run this test.

```bash
$ pytest tests/test_calculations.py

=========================== FAILURES ===========================
_________ test_average_rating_float_problem __________

    def test_average_rating_float_problem():
        # This test will FAIL!
        # The sum of 0.1 + 0.2 is not exactly 0.3
>       assert 0.1 + 0.2 == 0.3
E       assert (0.1 + 0.2) == 0.3
E        +  where (0.1 + 0.2) = 0.30000000000000004
===================== 1 failed in 0.12s =====================
```

Pytest's excellent assertion introspection shows us the problem clearly. The result of `0.1 + 0.2` is `0.30000000000000004`. This tiny difference causes the test to fail. This is the pitfall we need to avoid.

### The Solution: `pytest.approx`

Pytest provides a brilliant and simple solution for this: the `pytest.approx` helper. It allows you to assert that a number is "close enough" to an expected value, within a certain tolerance.

Let's fix our test and write a more realistic one for our function.

```python
# tests/test_calculations.py
import pytest
import numpy as np
from calculations import calculate_average_rating

def test_average_rating_simple():
    assert calculate_average_rating([1, 2, 3]) == pytest.approx(2.0)

def test_average_rating_with_floats():
    # This test now PASSES
    ratings = [0.1, 0.2, 0.3] # np.mean is 0.2
    assert calculate_average_rating(ratings) == pytest.approx(0.2)

def test_another_float_example():
    # The average of 1/3 and 2/3 is 0.5
    ratings = [1/3, 2/3]
    assert calculate_average_rating(ratings) == pytest.approx(0.5)
```

By wrapping the expected value in `pytest.approx()`, you're changing the assertion from "is exactly equal to" to "is approximately equal to". Pytest handles the tolerance logic for you.

### Using `pytest.approx` with NumPy and Pandas

The real power of `pytest.approx` is that it works seamlessly with collections like lists, NumPy arrays, and even Pandas Series/DataFrames. This makes it indispensable for data science testing.

Let's imagine a function that normalizes data (scales it between 0 and 1).

```python
# calculations.py
import numpy as np

def calculate_average_rating(ratings):
    if not ratings:
        return 0.0
    return np.mean(ratings)

def normalize_data(data: np.ndarray) -> np.ndarray:
    """Scales data to a [0, 1] range."""
    min_val = data.min()
    max_val = data.max()
    if max_val == min_val:
        return np.zeros_like(data, dtype=float)
    return (data - min_val) / (max_val - min_val)
```

Now, we can test it directly on a NumPy array.

```python
# tests/test_calculations.py
import pytest
import numpy as np
from calculations import calculate_average_rating, normalize_data

# ... (previous tests) ...

def test_normalize_data():
    input_data = np.array([10, 20, 30, 40, 50])
    expected_data = np.array([0.0, 0.25, 0.5, 0.75, 1.0])

    actual_data = normalize_data(input_data)

    # Use pytest.approx with a NumPy array
    assert actual_data == pytest.approx(expected_data)

def test_normalize_data_with_floats():
    input_data = np.array([0.1, 0.2, 0.3, 0.4, 0.5])
    # The normalization involves division, which will introduce float inaccuracies
    expected_data = np.array([0.0, 0.25, 0.5, 0.75, 1.0])

    actual_data = normalize_data(input_data)

    assert actual_data == pytest.approx(expected_data)
```

Whenever your tests involve floating-point numbers, your default should be to use `pytest.approx`. It makes your tests robust against tiny, irrelevant numerical differences and focuses them on what matters: whether the calculation is fundamentally correct.

## Testing Machine Learning Models

## Testing Machine Learning Models

This is where we truly depart from traditional software testing. We cannot assert that a model's prediction is "correct" in an absolute sense. Instead, we test the model's contract, its behavior, and its properties. Think of it like testing a car: you don't test that it can win a specific race, but you do test that the brakes work, the steering turns the wheels, and the engine doesn't explode.

Let's assume we have a simple classification model trained to identify spam emails. For testing, we don't want to re-train it every time. We'll assume we have a pre-trained model object that we can load.

Our tests should answer questions like:
-   Does the `predict` method run without errors? (A smoke test)
-   Does it return predictions in the format I expect?
-   Are the predictions reasonable? (e.g., probabilities are between 0 and 1)
-   Is the model deterministic if randomness is controlled?

Here's a simple example using a dummy model for demonstration.

```python
# model.py
import numpy as np

class SpamClassifier:
    """A dummy model for demonstration purposes."""
    def __init__(self, threshold=0.7):
        self.threshold = threshold

    def predict_proba(self, inputs: list[str]) -> np.ndarray:
        """
        Predicts the probability of each input being spam.
        A real model would use embeddings and a neural network.
        We'll just use a simple heuristic.
        """
        # A simple heuristic: if 'buy now' is in the text, high spam probability
        probabilities = []
        for text in inputs:
            if "buy now" in text.lower():
                probabilities.append(0.95)
            elif "hello" in text.lower():
                probabilities.append(0.1)
            else:
                probabilities.append(0.5)
        return np.array(probabilities)

    def predict(self, inputs: list[str]) -> np.ndarray:
        """Predicts a class label (0 for not-spam, 1 for spam)."""
        probabilities = self.predict_proba(inputs)
        return (probabilities >= self.threshold).astype(int)

# In a real scenario, you would load a saved model, e.g., with joblib or pickle
# For this example, we'll just instantiate it.
model = SpamClassifier()
```

Now, let's write tests for this model's behavior, not its accuracy on a large dataset.

```python
# tests/test_model.py
import pytest
import numpy as np
from model import SpamClassifier

@pytest.fixture
def spam_model():
    """Provides a trained SpamClassifier model instance."""
    return SpamClassifier(threshold=0.7)

def test_model_smoke_test(spam_model):
    """
    A simple smoke test to ensure predict() runs without crashing.
    """
    test_input = ["Hello, this is a friendly email."]
    try:
        spam_model.predict(test_input)
    except Exception as e:
        pytest.fail(f"Model prediction failed with an exception: {e}")

def test_prediction_output_shape(spam_model):
    """
    The number of predictions should match the number of inputs.
    """
    test_input = [
        "Hello friend",
        "URGENT: buy now for a big discount!",
        "Meeting at 3 PM"
    ]
    predictions = spam_model.predict(test_input)
    assert len(predictions) == len(test_input)

def test_prediction_output_type_and_values(spam_model):
    """
    Predictions should be integers (0 or 1) for classification.
    """
    test_input = ["Sample email 1", "Another one with buy now"]
    predictions = spam_model.predict(test_input)

    # Check data type
    assert predictions.dtype == int

    # Check that all values are either 0 or 1
    assert np.all(np.isin(predictions, [0, 1]))

def test_predict_proba_output_range(spam_model):
    """
    Predicted probabilities must be between 0 and 1.
    """
    test_input = ["A", "B", "C"]
    probabilities = spam_model.predict_proba(test_input)

    assert np.all((probabilities >= 0) & (probabilities <= 1))

def test_model_known_cases(spam_model):
    """
    Test the model on a few examples where the outcome is obvious.
    This checks the basic logic.
    """
    spam_email = ["URGENT call now to claim your prize, buy now!"]
    not_spam_email = ["Hi Bob, just confirming our meeting. Hello!"]

    assert spam_model.predict(spam_email)[0] == 1
    assert spam_model.predict(not_spam_email)[0] == 0
```

These tests establish a contract for our model. They don't care if the model is 99% accurate or 60% accurate. They care that it behaves as expected: it doesn't crash, it returns data in the correct shape and format, and its outputs are within a valid range. These are the kinds of tests that prevent bugs in your production machine learning systems.

## Fixtures for Data Pipelines

## Fixtures for Data Pipelines

As we've seen, data science tests require... well, data. You might be tempted to create this data inside every single test function.

Let's look at the "wrong way" first.

```python
# tests/test_pipeline_bad.py
import pandas as pd
from data_processing import preprocess_housing_data
from model import SpamClassifier # Pretend this is a housing model

def test_preprocessing_again():
    # Repetitive setup
    raw_data = {'price': ['$750,000'], 'year_built': [2005]}
    input_df = pd.DataFrame(raw_data)
    # ... assertions ...

def test_model_prediction_on_processed_data():
    # Repetitive setup again!
    raw_data = {'price': ['$750,000'], 'year_built': [2005]}
    input_df = pd.DataFrame(raw_data)
    processed_df = preprocess_housing_data(input_df)
    # model = HousingPriceModel()
    # ... assertions ...
```

This is repetitive and hard to maintain. If you need to add a new column to your test data, you have to change it in multiple places. This is exactly the problem fixtures were designed to solve (as we learned in Chapter 4).

By using fixtures, we can define our data and processing steps once and reuse them across many tests. This is especially useful for creating a mini-pipeline within our test suite.

Let's create a `conftest.py` file in our `tests/` directory to hold our shared fixtures.

```python
# tests/conftest.py
import pytest
import pandas as pd
from datetime import datetime

from data_processing import preprocess_housing_data
from model import SpamClassifier # Using this as a placeholder for a housing model

@pytest.fixture(scope="module")
def raw_housing_data() -> pd.DataFrame:
    """Fixture for raw housing data, loaded once per module."""
    print("\n(Creating raw_housing_data fixture)")
    data = {
        'price': ['$750,000', '$1,200,000', '$450,000', None],
        'year_built': [2005, 2018, 1990, 2010],
        'sq_ft': [2200, 3100, 1500, 1800]
    }
    return pd.DataFrame(data)

@pytest.fixture(scope="module")
def processed_housing_data(raw_housing_data: pd.DataFrame) -> pd.DataFrame:
    """
    Fixture that depends on raw_housing_data and provides cleaned data.
    """
    print("\n(Creating processed_housing_data fixture)")
    return preprocess_housing_data(raw_housing_data)

@pytest.fixture(scope="module")
def trained_model():
    """
    Fixture to provide a 'trained' model. In a real scenario, this might
    load a model from a file. Here, we just instantiate it.
    """
    print("\n(Creating trained_model fixture)")
    # In a real test suite, this would be your actual model,
    # e.g., a trained RandomForestRegressor.
    return SpamClassifier() # Placeholder
```

We've created a chain of dependencies: `processed_housing_data` depends on `raw_housing_data`. By setting `scope="module"`, we ensure this potentially expensive data loading and processing happens only *once* when we run the tests in a file, not before every single test. The `print` statements will help us see this in action.

Now, our test file becomes incredibly clean and readable. Each test just asks for the specific data it needs.

```python
# tests/test_pipeline_good.py
import pandas as pd
from datetime import datetime

def test_data_processing_output(processed_housing_data: pd.DataFrame):
    """Test the output of the processing step using a fixture."""
    # The setup is now completely handled by the fixture.
    # We can focus purely on the assertions.
    assert not processed_housing_data.isnull().values.any()
    assert 'house_age' in processed_housing_data.columns
    assert processed_housing_data['price'].dtype == 'float64'

def test_model_on_processed_data(trained_model, processed_housing_data: pd.DataFrame):
    """Test the model's behavior on clean data."""
    # This test needs both the model and the data.
    # Pytest provides both fixtures automatically.
    
    # For this placeholder, we need to convert the DataFrame to a list of strings
    # A real housing model would take the DataFrame directly.
    test_input = processed_housing_data['price'].astype(str).tolist()
    
    predictions = trained_model.predict(test_input)
    assert len(predictions) == len(processed_housing_data)
```

Let's run these tests with the `-s` (to show print statements) and `-v` (verbose) flags to see how the fixtures are executed.

```bash
$ pytest -sv tests/test_pipeline_good.py

=========================== test session starts ===========================
...
collected 2 items

tests/test_pipeline_good.py::test_data_processing_output
(Creating raw_housing_data fixture)
(Creating processed_housing_data fixture)
PASSED
tests/test_pipeline_good.py::test_model_on_processed_data
(Creating trained_model fixture)
PASSED

============================ 2 passed in 0.15s ============================
```

Notice the output. The data fixtures are created only once at the beginning, and the model fixture is created when it's first needed. Both are then reused for subsequent tests in the module. This pattern is the key to writing efficient, scalable, and maintainable tests for complex data pipelines.

## Testing Visualization Code

## Testing Visualization Code

Testing visual output is notoriously difficult. Do you save a "golden" image of the plot and compare it pixel-by-pixel to the new output? This is brittle; a tiny change in a dependency like Matplotlib could alter rendering slightly and break all your tests.

A much more robust strategy is to **test the data, not the pixels**. We should verify that our visualization function is called with the correct data and settings. We can trust that if we give Matplotlib the right numbers, it will draw the right chart.

This is a perfect use case for mocking, which we covered in Chapter 8. We can use `unittest.mock.patch` to intercept the call to the plotting function (e.g., `matplotlib.pyplot.bar`) and inspect the arguments it received.

Let's create a simple visualization function.

```python
# visualization.py
import pandas as pd
import matplotlib.pyplot as plt

def plot_average_price_by_year(df: pd.DataFrame):
    """
    Generates a bar plot of the average house price for each year built.
    """
    if df.empty:
        return None # Do nothing if no data

    avg_price_by_year = df.groupby('year_built')['price'].mean()

    plt.figure(figsize=(10, 6))
    plt.bar(
        x=avg_price_by_year.index,
        height=avg_price_by_year.values
    )
    plt.xlabel("Year Built")
    plt.ylabel("Average Price ($)")
    plt.title("Average Housing Price by Year Built")
    plt.tight_layout()
    # In a real app, you might call plt.savefig() or plt.show()
    # For testing, we'll just let the function end here.
    return plt.gca() # Return the current axes object for inspection
```

Now, let's write a test. We will "patch" `matplotlib.pyplot.bar`. This replaces the real `bar` function with a "MagicMock" object that records how it was called.

```python
# tests/test_visualization.py
import pytest
import pandas as pd
import numpy as np
from unittest.mock import patch

from visualization import plot_average_price_by_year

@pytest.fixture
def viz_data() -> pd.DataFrame:
    """Sample data for visualization tests."""
    data = {
        'price': [500000, 600000, 700000, 800000],
        'year_built': [2010, 2010, 2012, 2012],
    }
    return pd.DataFrame(data)

# The patch decorator intercepts the call to 'visualization.plt.bar'
@patch('visualization.plt.bar')
def test_plot_average_price_by_year_data(mock_bar, viz_data):
    """
    Test that plt.bar is called with the correctly aggregated data.
    """
    # Call the function that we are testing
    plot_average_price_by_year(viz_data)

    # Assert that our mock was called exactly once
    mock_bar.assert_called_once()

    # The arguments are available in mock_bar.call_args
    # It's a tuple of (args, kwargs)
    args, kwargs = mock_bar.call_args

    # The data is passed as keyword arguments 'x' and 'height'
    x_values = kwargs.get('x')
    height_values = kwargs.get('height')

    # Expected data after groupby and mean aggregation
    # Year 2010: mean(500000, 600000) = 550000
    # Year 2012: mean(700000, 800000) = 750000
    expected_x = np.array([2010, 2012])
    expected_height = np.array([550000.0, 750000.0])

    # Use numpy's testing utilities for array comparison
    np.testing.assert_array_equal(x_values, expected_x)
    np.testing.assert_array_equal(height_values, expected_height)

@patch('visualization.plt.title')
@patch('visualization.plt.xlabel')
@patch('visualization.plt.ylabel')
def test_plot_labels_and_title(mock_ylabel, mock_xlabel, mock_title, viz_data):
    """Test that the plot labels and title are set correctly."""
    plot_average_price_by_year(viz_data)

    mock_title.assert_called_once_with("Average Housing Price by Year Built")
    mock_xlabel.assert_called_once_with("Year Built")
    mock_ylabel.assert_called_once_with("Average Price ($)")
```

This approach is far superior to image comparison:
-   **It's fast:** It doesn't require any actual image rendering.
-   **It's robust:** It's immune to changes in library versions that affect rendering.
-   **It's precise:** It tests the core logic of the function—the data aggregation—which is what you, the developer, are responsible for.

By testing the inputs to the plotting library, you verify that your code is doing its job correctly without getting bogged down in the fragile and complex world of visual testing.
