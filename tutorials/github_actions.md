# Continues Integration (CI) with GitHub Actions

Continues Integration (CI) is a software development practice where developers frequently integrate their code changes into the same branch, that later will be used to deploy the application.
Each integration is verified by an **automated tests** to detect integration errors as quickly as possible.


To achieve that, we need an **automation platform** - some server that manages and execute the automations, with nice UI to monitor the results, and that can be easily integrated with our code repository.

For that, we will use a platform which is part of GitHub, called **GitHub Actions**.

GitHub Actions is a continuous integration and continuous delivery (CI/CD) platform that allows you to automate your build, test, and deployment pipelines.

Let's add a CI workflow to the YOLO service we've been working on.

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


A **workflow** is a configurable automated process that will run one or more **jobs**.

Workflows are defined as a [YAML](https://learnxinyminutes.com/docs/yaml/) file located in the source code repo, in the `.github/workflows` directory in a repository.
The workflows can be configured to be running on different events in your repo, such a code push, by periodic schedule, or manually.

For GitHub to discover any GitHub Actions workflows in your repository, you must save the workflow files in a directory called `.github/workflows`.
You can give the workflow file any name you like, but you must use `.yml` or `.yaml` as the file name extension.

In the above workflow, there is one job called `test`, which runs a series of steps to set up the Python environment, install dependencies, and run tests using `pytest`.
Step are being executed one after another, and if one of the steps fails, the job will stop executing and be marked as failed.

## Git workflow

Let's see a standard feature-branch workflow used in professional software development.

To demonstrate it, we will use a simple example.
Let's say you have a task to return the prediction time in the `/predict` response. You would follow these steps:

#### Step 1 - Create a feature branch

From `main`, create a new branch for this change:

```bash
git checkout -b prediction_time
```

#### Step 2 - Implement the change

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

#### Step 3 - Write a test

To ensure your feature was implemented properly, you have to write tests. Copy the test code below under `services/yolo/tests/test_prediction_time.py`:

```python
import os
import unittest
import tempfile
from fastapi.testclient import TestClient
import app as app_module
from app import app, init_db

TEST_IMAGE = os.path.join(os.path.dirname(__file__), "data", "beatles.jpeg")


class TestPredictionTime(unittest.TestCase):
    def setUp(self):
        _, app_module.DB_PATH = tempfile.mkstemp(suffix=".db")
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


#### Step 4 - Commit, push, and open a PR

1. Commit and push your changes using the VS Code Source Control panel.
2. Open a Pull Request on GitHub from `prediction_time` → `main`.
3. Wait for the CI checks to pass. If they fail, check the logs, fix the issue, and push again until they pass.
4. Merge the PR into `main`.