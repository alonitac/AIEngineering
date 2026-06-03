# The Yolo Service - new endpoints and API testing

In this task you will implement missing API endpoints for the YOLO service and write tests that achieve 100% code coverage.


## Git Workflow Walkthrough

This section walks you through the standard feature-branch workflow used in professional software development.

To demonsrate it, we will use a simple example.
Let's say you have a task to return the prediction time in the `/predict` response. You would follow these steps:

### Step 1 - Create a feature branch

From `main`, create a new branch for this change:

```bash
git checkout -b prediction_time
```

### Step 2 - Implement the change

Apply the following diff to `services/yolo/app.py`:

```diff
+ import time

 @app.post("/predict")
 def predict(file: UploadFile = File(...)):
+    start_time = time.time()

     ext = os.path.splitext(file.filename)[1]
     uid = str(uuid.uuid4())
     original_path = os.path.join(UPLOAD_DIR, uid + ext)
     predicted_path = os.path.join(PREDICTED_DIR, uid + ext)

     with open(original_path, "wb") as f:
         shutil.copyfileobj(file.file, f)

     results = model(original_path, device="cpu")

     annotated_frame = results[0].plot()
     annotated_image = Image.fromarray(annotated_frame)
     annotated_image.save(predicted_path)

     save_prediction_session(uid, original_path, predicted_path)

     detected_labels = []
     for box in results[0].boxes:
         label_idx = int(box.cls[0].item())
         label = model.names[label_idx]
         score = float(box.conf[0])
         bbox = box.xyxy[0].tolist()
         save_detection_object(uid, label, score, bbox)
         detected_labels.append(label)

+    processing_time = round(time.time() - start_time, 2)

     return {
         "prediction_uid": uid,
         "detection_count": len(results[0].boxes),
         "labels": detected_labels,
+        "time_took": processing_time
     }
```

### Step 3 - Write a test

To ensure your feature was implemented properly, you have to write tests. Copy the test code below under `services/yolo/tests/test_prediction_time.py`:

```python
import os
import unittest
from fastapi.testclient import TestClient
import app as app_module
from app import app, init_db

TEST_IMAGE = os.path.join(os.path.dirname(__file__), "data", "beatles.jpeg")


class TestPredictionTime(unittest.TestCase):
    def setUp(self):
        app_module.DB_PATH = ":memory:"
        init_db()
        self.client = TestClient(app)

    def test_predict_includes_processing_time(self):
        with open(TEST_IMAGE, "rb") as f:
            response = self.client.post(
                "/predict",
                files={"file": ("beatles.jpeg", f, "image/jpeg")}
            )

        self.assertEqual(response.status_code, 200)
        data = response.json()
        self.assertIn("time_took", data)
        self.assertIsInstance(data["time_took"], (int, float))
        self.assertGreaterEqual(data["time_took"], 0)
```

Run the test locally to confirm it passes:

```bash
pytest tests/test_prediction_time.py
```

### Step 4 - Add a CI workflow

Create `.github/workflows/test.yaml` in the **root** of the repository:

```yaml
name: Run Tests

on:
  pull_request:
    branches:
      - main

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v4
        with:
          python-version: '3.12'

      - name: Install dependencies
        run: |
          pip install -r services/yolo/torch-requirements.txt
          pip install -r services/yolo/requirements.txt .

      - name: Run tests
        run: pytest tests/
        working-directory: services/yolo
```

### Step 5 - Commit, push, and open a PR

1. Commit and push your changes using the VS Code Source Control panel.
2. Open a Pull Request on GitHub from `prediction_time` → `main`.
3. Wait for the CI checks to pass. If they fail, check the logs, fix the issue, and push again until they pass.
4. Merge the PR into `main`.


## Implement missing endpoints

The YOLO service `README.md` documents several API endpoints that **are not yet implemented** in `app.py`.

Your job is to implement all of them. For each endpoint, follow the same workflow as above: create a feature branch, implement the endpoint, write tests, open a PR, pass CI, and merge.

### Endpoints to implement

#### `GET /predictions/label/{label}`

Return all prediction sessions that contain at least one detected object with the given label (e.g. `"person"`, `"car"`).

- Response `200 OK` - always returns a list (empty if no matches):

For `GET /predictions/label/person`:

```json
[
  {
    "uid": "abc-123",
    "timestamp": "2024-01-01 12:00:00",
    "detection_objects": [
      { "id": 1, "label": "person", "score": 0.91, "box": "[10, 20, 100, 200]" }
    ]
  }
]
```

- If `label` is an empty string, return `400` with `detail: "Label cannot be empty"`.

#### `GET /predictions/score/{min_score}`

Return all detection objects whose confidence score is greater than or equal to `min_score`.

`min_score` is a float path parameter between `0.0` and `1.0`.

- Response `200 OK` - always returns a list (empty if no matches):

For `GET /predictions/score/0.5`:

```json
[
  { "id": 1, "prediction_uid": "abc-123", "label": "person", "score": 0.91, "box": "[10, 20, 100, 200]" }
]
```

- If `min_score` is not between `0.0` and `1.0`, return `400` with `detail: "min_score must be between 0.0 and 1.0"`.


#### Report coverage in GitHub via Codecov

1. Make sure your tests uses `pytest-cov` to generate a coverage report in XML format.

1. Open an account on [Codecov](https://codecov.io/) and link it to your GitHub repository.
2. Add an upload step to your workflow:

```diff
      - name: Run tests with coverage
        run: pytest tests/
        working-directory: services/yolo

+     - name: Upload coverage to Codecov
+       uses: codecov/codecov-action@v3
+       with:
+         file: services/yolo/coverage.xml
+         fail_ci_if_error: true
```


Write enough tests to achieve **100% code coverage** of all **endpoint functions**.

# Good


