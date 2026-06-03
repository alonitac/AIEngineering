# The Yolo Service - new endpoints and API testing

In this task you will implement missing API endpoints for the YOLO service and write tests that achieve 100% code coverage.


## Implement missing endpoints

The YOLO service `README.md` documents several API endpoints that **are not yet implemented** in `app.py`.

For each endpoint detailed below, and for the rest of your life as a developer (unless otherwise instructed by your boss), you MUST follow the Git workflow as described in [CI with GitHub Actions tutorial](../tutorials/github_actions.md): create a feature branch, implement the endpoint, write tests, open a PR, pass CI, and merge.

Never implement code directly on `main` or merge a PR without a passing CI build, even if it's just a small change and you 100% sure it works.


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

# Good Luck!


