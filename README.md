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
- `clone-source`, `flake8`, `build-image`, and `deploy-app`: check out the
  repository, lint the code, build the container, and deploy the tested image.

The [pipeline](.tekton/pipeline.yml) runs these stages in order:

```text
cleanup → clone → lint → test → build → deploy
```

The build uses OpenShift's binary Docker build with the tested source from
the shared workspace. Deployment selects the resulting image by digest and
waits for the application's health check to pass.

## Run in the OpenShift lab

The lab needs OpenShift Pipelines with the `tekton.dev/v1beta1` API, a default
storage class, and the OpenShift build service and internal image registry.
Use the terminal and assigned OpenShift project in the Skills Network lab.

Clone the project if it is not already present:

```bash
git clone https://github.com/felonardo/ci-cd-final-project.git
cd ci-cd-final-project
```

Check the logged-in account and assigned project, then create the resources:

```bash
oc whoami
oc project
oc apply -f .openshift/resources.yml
oc apply -f .tekton/tasks.yml
oc apply -f .tekton/pipeline.yml
oc create -f .tekton/pipelinerun.yml
```

The resources include a 1 GiB PVC named `pipeline-pvc` using the
`skills-network-learner` storage class, a service account with namespace-scoped
build/deployment permissions, an ImageStream, a BuildConfig,
a Deployment, a Service, and a Route. The Deployment starts with zero replicas;
the deploy task starts one replica after the image build succeeds.

Monitor the run in **Pipelines → ci-cd-final-project → PipelineRuns** or use:

```bash
oc get pipelineruns
oc get taskruns
oc get pvc pipeline-pvc
oc get pods
```

Start another run with `oc create -f .tekton/pipelinerun.yml` after the previous
run finishes. Runs share the same workspace and application.

After the pipeline succeeds, request the application and save its actual logs:

```bash
APP_HOST=$(oc get route ci-cd-final-project -o jsonpath='{.spec.host}')
curl --fail "http://$APP_HOST/"
curl --fail "http://$APP_HOST/health"
oc logs deployment/ci-cd-final-project --timestamps=true > oc-pipelines-app-logs.txt
```

## Submission evidence

| Requirement | File or evidence |
| --- | --- |
| Project name | `README.md` |
| GitHub Actions lint and unit tests | `.github/workflows/workflow.yml` |
| Tekton cleanup and nose tasks | `.tekton/tasks.yml` |
| PVC details, showing Bound status | `oc-pipelines-console-pvc-details.png` |
| Successful GitHub Actions output with every step | Text from the successful Actions run |
| OpenShift pipeline details | `oc-pipelines-oc-final.png` |
| Successful OpenShift pipeline run | `oc-pipelines-oc-green.png` |
| Application logs | Text from `oc-pipelines-app-logs.txt` |

Capture screenshots and logs from the actual completed runs in the lab.

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
