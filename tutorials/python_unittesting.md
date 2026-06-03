# Unit Testing in Python


**Unit testing** is a software testing method where individual units of code (typically functions or methods) are tested in isolation. The key principles are:

## Basic Example

Let's say we want to test the following simple calculator functions:

```python
import unittest

def add(a, b):
    return a + b

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b
```

To do so, we define the `TestCalculator` test case class. 

A **Test Case** is a single class that verifies a specific behavior of your code.

```python
class TestCalculator(unittest.TestCase):
    
    def test_add_positive_numbers(self):
        result = add(3, 5)
        self.assertEqual(result, 8)
    
    def test_add_negative_numbers(self):
        result = add(-1, -1)
        self.assertEqual(result, -2)
    
    def test_divide_normal(self):
        result = divide(10, 2)
        self.assertEqual(result, 5.0)
    
    def test_divide_by_zero(self):
        with self.assertRaises(ValueError):
            divide(10, 0)

if __name__ == '__main__':
    unittest.main()
```

We can execute the test simply by:

```bash
python -m unittest test_calculator.py
```

This will run all the test methods (functions start with `test_`) defined in the `TestCalculator` class. 

As can be seen, in each test method, we use **assertions** to verify that the output of our functions matches the expected results.
Assertions are methods that check if a condition is true. If the assertion fails, the test fails.


By default, **order** of test execution is not guaranteed.

> [!TIP]
> #### Pytest
> 
> We can also use `pytest`, which is a more powerful testing framework with additional features.
> 
> `pytest` can be used not only to execute tests, but also to write them. In this course, you may use either `pytest` or Python's built-in `unittest` module, as long as you understand what your tests are doing and can explain them.
> 
> To run tests with `pytest`, simply execute:
> ```bash
> pytest test_calculator.py
> ```


## Setup and Teardown

Setup and teardown methods allow you to prepare **test conditions** before tests run and clean up afterwards.

```python
import unittest

class TestListLength(unittest.TestCase):
    
    def setUp(self):
        """Called before each test method"""
        self.test_list = [1, 2, 3]
        print("setUp: Creating test_list")
    
    def tearDown(self):
        """Called after each test method"""
        self.test_list = None
        print("tearDown: Cleaning up test_list")
    
    def test_list_length(self):
        self.assertEqual(len(self.test_list), 3)
    
    def test_list_append(self):
        self.test_list.append(4)
        self.assertEqual(len(self.test_list), 4)
```

## How to write a good unittest

It takes practice to write good unit tests. 
You can take a look at almost any open-source project to see examples of unit tests.

But here are some best practices to follow when writing unit tests:

#### Tests are isolated and do not depend on each other

Bad:

```python
class TestCounter(unittest.TestCase):
    counter = 0

    def test_increment(self):
        TestCounter.counter += 1
        self.assertEqual(TestCounter.counter, 1)

    def test_increment_again(self):
        TestCounter.counter += 1
        self.assertEqual(TestCounter.counter, 1)  # order-dependent
```

#### Tests are **deterministic**

Same input → same result every time. 

Bad:


```python
class TestRandom(unittest.TestCase):

    def test_random_number(self):
        self.assertEqual(random.randint(1, 10), 5)  # flaky
```

Good:

```python
class TestRandom(unittest.TestCase):

    @patch("random.randint")
    def test_random_number(self, mock_randint):
        mock_randint.return_value = 5
        self.assertEqual(random.randint(1, 10), 5)
```

#### Tests run fast

Bad:

```python
class TestTimeout(unittest.TestCase):

    def test_retry_timeout(self):
        attempts = 0
        while attempts < 10:
            # simulate network call that should fail
            
            time.sleep(2)  # sleep before the next try
            attempts += 1  # simulate retry loop
        self.assertEqual(attempts, 10)
```

Good:


```python
class TestTimeout(unittest.TestCase):

    @patch("time.sleep")
    def test_retry_timeout(self, _):
        attempts = 0
        while attempts < 10:
            # simulate network call that should fail

            time.sleep(2)      # mocked → no real delay
            attempts += 1      # retry loop
        self.assertEqual(attempts, 10)
```


#### Tests do not rely on external systems

During tests you cannot call the real YOLO model - it is slow, and it's not the point of the test. Use `patch` to replace it with a fake.

Bad:

```python
def test_predict(client):
    with open("tests/data/beatles.jpeg", "rb") as f:
        response = client.post("/predict", files={"file": f})  # loads real YOLO model!
    assert response.status_code == 200
```

Good:

```python
from unittest.mock import MagicMock, patch

def test_predict(client):
    fake_result = MagicMock()
    fake_result.boxes = []
    fake_result.plot.return_value = MagicMock()

    with patch("app.model", return_value=[fake_result]) as mock_model:
        mock_model.names = {}
        with open("tests/data/beatles.jpeg", "rb") as f:
            response = client.post("/predict", files={"file": f})

    assert response.status_code == 200
```

#### Edge cases and boundary conditions are covered

Bad:

```python

def divide(a, b):
    return a / b

class TestDivide(unittest.TestCase):

    def test_divide_normal(self):
        self.assertEqual(divide(10, 2), 5)
```

Good:

```python
def divide(a, b):
    return a / b

class TestDivide(unittest.TestCase):

    def test_divide_normal(self):
        self.assertEqual(divide(10, 2), 5)

    def test_divide_by_zero(self):
        with self.assertRaises(ZeroDivisionError):
            divide(10, 0)

    def test_divide_negative(self):
        self.assertEqual(divide(-10, 2), -5)
        self.assertEqual(divide(10, -2), -5)
```


# API Testing

API testing is a **functional testing** test type that ensures that the API endpoints function correctly, handle errors, and meet the expected behaviour.

In this tutorial, we will explore how to test APIs built with FastAPI using Python's built-in `unittest` framework and the `TestClient` object.

But before, let's meet some important concepts in testing methodology:

#### Test analysis

Test analysis, simply said, is the activity of deciding **what to test**.

Choosing to test everything is usually impractical, due to time and resource constraints.
Hence, we need to analyze the system under test and decide which parts are most critical and require testing.

When testing APIs, writing tests that verify the functionality of the API endpoints is the most straightforward choice, as they are the main interface for users to interact with the application.

#### Test design

Test design is the process of deciding the strategy we will use to test the system, the **how to test**.

- Should we test our API with a real database?
- Should we call the real YOLO model during the test?
- Should we run a real app instance or use a "test client" that simulates app behaviour?
- Should we verify that data was written to the DB?

Currently, our strategy is to use a real SQLite database (in a temporary directory) during the tests, **mock the YOLO model**, and use FastAPI's `TestClient` to simulate app behaviour without running a real server.

#### Success criteria

Success criteria is a **clear definition** of when you have enough confidence that the app works correctly.

Usually it is written in terms of **coverage**:

- **Code coverage** – lines, branches, or paths of code executed by tests
- **Requirements coverage** – all functional or business requirements are tested
- **Risk coverage** – areas with high complexity, critical business impact, or history of defects

In our case, we will focus on **API coverage** - making sure that all API endpoints and their functionality are tested.


## API testing with FastAPI

### Install dependencies

To use `TestClient`, first install `httpx` in your venv:

```bash
pip install httpx
```

### Example

Here is a simplified version of our YOLO app and a test for it:

```python
# app.py

from fastapi import FastAPI

app = FastAPI()

@app.get("/health")
def health():
    return {"status": "ok"}
```

Under `tests/test_api.py`, we write a test for the above endpoint:

```python
import unittest
from fastapi.testclient import TestClient
from app import app


class TestHealth(unittest.TestCase):
    def setUp(self):
        self.client = TestClient(app)

    def test_health(self):
        response = self.client.get("/health")
        self.assertEqual(response.status_code, 200)
        self.assertEqual(response.json(), {"status": "ok"})
```

In this test:

1. We create a `TestCase` class and initialize the `TestClient` in `setUp`, which runs **before each test method**.
2. We define `test_health` which sends a GET request to `/health` and checks that the response status code is `200` and the body matches the expected JSON.

#### Tests do not rely on external systems

During tests you cannot call the real YOLO model - it is slow, and out of scope for unit tests. Use `patch` to replace it with a fake.


```python
import unittest
from unittest.mock import MagicMock, patch
from fastapi.testclient import TestClient
from app import app


class TestPredict(unittest.TestCase):
    def setUp(self):
        self.client = TestClient(app)

    @patch("app.Image")   # prevent PIL from processing a fake frame
    @patch("app.model")   # prevent the real YOLO model from running
    def test_predict(self, mock_model, mock_image):
        fake_result = MagicMock()
        fake_result.boxes = []   # no detections - keeps the test simple
        mock_model.return_value = [fake_result]
        mock_model.names = {}

        with open("tests/data/beatles.jpeg", "rb") as f:
            response = self.client.post("/predict", files={"file": f})

        body = response.json()
        self.assertEqual(response.status_code, 200)
        self.assertEqual(body["detection_count"], 0)
        self.assertEqual(body["labels"], [])
        self.assertIn("prediction_uid", body)
```

> Note: `@patch` decorators are applied bottom-up, so `mock_model` maps to `@patch("app.model")` and `mock_image` maps to `@patch("app.Image")`.

#### Tests use an isolated database

The YOLO app writes prediction results to SQLite. Tests should use a **temporary database** so they don't pollute (or depend on) the real one. Override `DB_PATH` in `setUp` and initialize a fresh schema before each test:

```python
import unittest
import app as app_module
from app import app, init_db
from fastapi.testclient import TestClient


class TestPredict(unittest.TestCase):
    def setUp(self):
        app_module.DB_PATH = ":memory:"
        init_db()
        self.client = TestClient(app)
```

`":memory:"` tells SQLite to create a temporary database that lives only in RAM and is discarded when the connection closes - no files to clean up.


## Reporting

Good test reporting helps you and the product team to track code quality and see test results without being that technical. You'll set up automated test reporting that shows coverage percentages and test results in your pull requests.


1. Add the following testing dependencies to `services/yolo/requirements.txt`:
   ```
   pytest==7.4.0
   pytest-cov==4.1.0
   pytest-html==3.2.0
   ```

2. Create `services/yolo/pytest.ini`. 
   ```ini
   [pytest]
   testpaths = tests
   addopts = --cov=app --cov-report=html --cov-report=xml --cov-report=term-missing
   ```
   This tells pytest to:
   - Look for tests in the `tests/` directory.
   - Generate **code coverage reports** for the `app.py` module.

3. Run tests and see coverage in the terminal:
   ```bash
   pytest
   ```

4. To generate a detailed HTML report:
   ```bash
   coverage html
   ```
   This creates a `htmlcov/` directory. Open `htmlcov/index.html` in a browser to explore which lines are covered.


