# CI/CD Final Project: Counter Service

Final project for the Coursera course **CI/CD Tools and Practices**.

This project uses a Python Flask REST API to create, list, read, increment,
and delete named counters. Counters are stored in memory and reset when the
application restarts.

## Continuous integration

The [GitHub Actions workflow](.github/workflows/workflow.yml) runs on pushes
and pull requests to `main`, and can also be started manually. It checks out
the code, sets up Python 3.9, installs dependencies, lints with flake8, and
runs the nose unit tests with a minimum of 90% coverage.

Python 3.9 matches the course template and its nose dependencies.

## Run locally

```bash
python3.9 -m venv venv
source venv/bin/activate
python -m pip install --upgrade pip wheel
python -m pip install -r requirements.txt
flake8 service tests --count --select=E9,F63,F7,F82 --show-source --statistics
flake8 service tests --count --max-complexity=10 --max-line-length=127 --statistics
nosetests -v --with-spec --spec-color --with-coverage --cover-package=service --cover-min-percentage=90
gunicorn service:app --bind 0.0.0.0:8000
```

Open `http://localhost:8000/` for service information or `/health` for the
health check.

## OpenShift pipeline tasks

The [Tekton task definitions](.tekton/tasks.yml) provide:

- `cleanup`: removes previous checkout files, including hidden files, from
  the shared `source` workspace before a new checkout.
- `nose`: installs the project dependencies in an isolated environment and
  runs the same unit tests and coverage threshold used by GitHub Actions.

In the course's OpenShift project, register the tasks with:

```bash
oc apply -f .tekton/tasks.yml
```

Bind both tasks to the pipeline's shared persistent workspace. Run cleanup
before cloning the repository, and run nose after cloning and before building
and deploying the application.

## API endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| GET | `/` | Service information |
| GET | `/health` | Health check |
| GET | `/counters` | List counters |
| POST | `/counters/<name>` | Create a counter |
| GET | `/counters/<name>` | Read a counter |
| PUT | `/counters/<name>` | Increment a counter |
| DELETE | `/counters/<name>` | Delete a counter |

## License and attribution

Based on the Skills Network course template. Licensed under the
[Apache License 2.0](LICENSE).

© IBM Corporation 2023. All rights reserved.
