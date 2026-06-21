
# The Yolo Service - SQLAlchemy Refactor via Agent Skills

## Overview

In this task you'll refactor the YOLO service's database layer from raw SQLite to SQLAlchemy. Keep smiling, you won't write the refactoring code yourself. You'll write an **agent skill** that instructs the coding agent to do the entire refactor for you. The skill will be used for any future features that require API data layer changes.


## Introducing SQLAlchemy

SQLAlchemy is a Python library that lets you interact with databases using Python classes instead of raw SQL strings. It also makes your app **database agnostic** - your code will be run against SQLite in development and Postgres in production, just by changing an environment variable.

### Modeling the data

In SQLAlchemy, you write **Python classes** instead of **SQL tables**:

```python
# models.py

from sqlalchemy import Column, String, DateTime, Integer, Float
from sqlalchemy.ext.declarative import declarative_base
from datetime import datetime

Base = declarative_base()


class PredictionSession(Base):
    __tablename__ = 'prediction_sessions'

    uid = Column(String, primary_key=True)
    timestamp = Column(DateTime, default=datetime.utcnow)
    original_image = Column(String)
    predicted_image = Column(String)


class DetectionObject(Base):
    __tablename__ = 'detection_objects'

    id = Column(Integer, primary_key=True, autoincrement=True)
    prediction_uid = Column(String)
    label = Column(String)
    score = Column(Float)
    box = Column(String)
```

- **Model** - a Python class that maps to a database table
- **Class attribute** - a database column
- **Class instance** - a database row

As the models are defined **declaratively**, you don't need to manually create the tables. The `init_db()` function is no longer needed. Tables are created automatically.

### Database connection

```python
# db.py

import os
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker

DB_BACKEND = os.getenv("DB_BACKEND", "sqlite")
DB_USER = os.getenv("DB_USER", "user")
DB_PASSWORD = os.getenv("DB_PASSWORD", "pass")

if DB_BACKEND == "postgres":
    DATABASE_URL = f"postgresql://{DB_USER}:{DB_PASSWORD}@localhost/db"
else:
    DATABASE_URL = "sqlite:///./predictions.db"

engine = create_engine(
    DATABASE_URL,
    connect_args={"check_same_thread": False} if "sqlite" in DATABASE_URL else {},
)

SessionLocal = sessionmaker(bind=engine, autoflush=False, autocommit=False)

def get_db():
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()
```

### Database operations

**INSERT** - instead of:
```python
conn.execute("INSERT INTO prediction_sessions (uid, original_image) VALUES (?, ?)", (uid, path))
```

We do:

```python
row = PredictionSession(uid=uid, original_image=path)
db.add(row)
db.commit()
```

**SELECT** - instead of:

```python
result = conn.execute("SELECT * FROM prediction_sessions WHERE uid = ?", (uid,)).fetchone()
```

We do:

```python
result = db.query(PredictionSession).filter_by(uid=uid).first()
```

### FastAPI dependency injection

FastAPI has a built-in **dependency injection** system. Instead of calling `get_db()` manually, you declare it as a parameter and FastAPI wires it up automatically:

```python
from fastapi import Depends
from sqlalchemy.orm import Session
from db import get_db

@app.get("/predictions/{uid}")
def get_prediction(uid: str, db: Session = Depends(get_db)):
    result = db.query(PredictionSession).filter_by(uid=uid).first()
    ...
```

## Write the skill

Write a skill called `yolo-api-data-layer`. Here are the kinds of prompts that should activate it:

- "refactor the api to use sqlalchemy"
- "add an endpoint GET /predictions/recent that returns the 10 most recent sessions"
- "add a UserFeedback table to track user ratings per prediction"
- "write tests for the /predict endpoint"
- "the database layer doesn't follow our architectural design, fix it"
- "delete a prediction session and all its detection objects by uid"
- "add a column `processing_time_ms` to the prediction_sessions table"
- "make the database backend configurable so we can use postgres in production"

> [!NOTE]
> All existing endpoints, status codes, and response shapes must stay exactly the same after the refactor. 

> [!NOTE]
If you've already implemented the `yolo-api-tests` skill from a our class exercises, modify it to use the new model. 



## Write evals for your skill

A skill without [evals](https://agentskills.io/skill-creation/evaluating-skills) is a skill you can't trust. Write at least **5 eval cases** in `.agents/skills/data-layer/evals/evals.json`.

Each eval case needs:
- A `prompt` - something a developer might actually type
- An `expected_output` - a concrete, checkable description of what the agent should produce
- Assetions 

Make your expected outputs specific. "Creates models.py with SQLAlchemy models" is better than "does the refactor". Include things like: which files should be created or modified, what patterns should appear in the output, what should *not* appear (e.g. no raw SQL strings, no `import sqlite3`).


## Useful community skills

There are no community skills for SQLAlchemy or FastAPI (which is exactly why you're writing one). But there are well-maintained skills that might help you throughout this task:

- `obra/superpowers@writing-skills` - Writing or improving the `yolo-api-data-layer` skill itself
- `obra/superpowers@verification-before-completion` - Forces the agent to run tests and verify output before claiming it's done
- `anthropics/skills@webapp-testing` - Patterns for unit, integration, and end-to-end testing of web APIs


## Let the agent do the refactor

With your skill in place, open your coding agent (Cloud Code, Copilot, Codex, etc.) in **agent mode** and type:

```
refactor the api to work with sqlalchemy
```

Or 

```
use the yolo-api-data-layer skill to refactor the api to work with sqlalchemy
```

You can modfy the prompt a bit, but is must be that simple and short, the agent should be able to handle it. The skill you wrote will guide the agent through the entire refactor.

1. Let it run to completion.
2. After it finishes try to launch the app and see if it runs without errors.
3. Run the tests: `pytest tests/`. Did they all pass? Is there any regression in the coverage level? 
4. **DON'T** fix anything with your own hands or by ad-hoc prompts. If the agent's output is wrong, you need to fix the skill and re-run the evals until it passes.

> [!TIP]
> Use `git reset --hard` to undo the agent's changes and start over if you need to.

## Testing with PostgreSQL

Installing PostgreSQL directly on your machine is heavy - it runs as a Linux service, it's hard clean state, and it can conflict with other projects.
Docker lets you run a fully isolated PostgreSQL instance in a container that disappears the moment you stop it, with zero impact on your machine. We'll cover Docker properly later in the course.

Spin up a PostgreSQL instance:

```bash
docker run --rm -e POSTGRES_USER=user -e POSTGRES_PASSWORD=pass -e POSTGRES_DB=predictions -p 5432:5432 postgres
```

Then run the Yolo app with the Postgres backend:

```bash
export DB_BACKEND=postgres
export DB_USER=user
export DB_PASSWORD=pass
python app.py
```

Send a few requests with `curl` or Postman and confirm the app behaves the same as with SQLite.


# Good Luck!