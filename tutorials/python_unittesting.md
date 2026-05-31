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
