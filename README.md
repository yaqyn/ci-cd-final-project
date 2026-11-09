# CI/CD Tools and Practices Final Project Template

## Project: ci-cd-final-project

A Flask-based **Hit Counter Service** used to demonstrate an end-to-end CI/CD
workflow. The service exposes a small JSON API for creating, reading, updating,
listing, and deleting named counters, plus health and index endpoints.

The project is built from the official IBM Skills Network template
(`vselh-ci-cd-final-project-template`) and shows how the same application can be
continuously integrated with GitHub Actions and continuously delivered with a
Tekton pipeline on OpenShift.

## Application endpoints

| Method | Path | Description |
| --- | --- | --- |
| `GET` | `/` | Service information |
| `GET` | `/health` | Health check (`{"status": "OK"}`) |
| `GET` | `/counters` | List all counters |
| `POST` | `/counters/<name>` | Create a counter |
| `GET` | `/counters/<name>` | Read a counter |
| `PUT` | `/counters/<name>` | Increment a counter |
| `DELETE` | `/counters/<name>` | Delete a counter |

## CI/CD workflow overview

1. **Continuous Integration — GitHub Actions** (`.github/workflows/workflow.yml`)
   - Runs on every push to `main` and every pull request targeting `main`.
   - Runs inside the `python:3.9-slim` container on `ubuntu-latest`.
   - Checks out the repository with `actions/checkout@v3`.
   - Installs `requirements.txt`, then runs **flake8** static analysis and the
     **nose** unit test suite.
2. **Continuous Delivery — Tekton on OpenShift** (`.tekton/tasks.yml`)
   - `cleanup` clears the shared workspace, including hidden files.
   - `git-clone` fetches the application source.
   - `flake8` lints the source.
   - `nose` runs the unit tests.
   - `buildah` builds and pushes the container image.
   - `openshift-client` creates/updates the Deployment and exposes the app.
   - Pipeline definition: `.tekton/pipeline.yml`; storage: `.tekton/pvc.yml`.
3. **Deployment** — the Flask service runs under Gunicorn in a container. The
   application logs `S E R V I C E   R U N N I N G` on startup so the pipeline
   evidence can confirm the deployment is healthy.

## Setup and testing

### Prerequisites

- Python 3.9 (the CI workflow and container image use 3.9)
- Git

### Local setup

```bash
git clone https://github.com/<your-account>/ci-cd-final-project.git
cd ci-cd-final-project

python3.9 -m venv venv
source venv/bin/activate
python -m pip install --upgrade pip wheel
pip install -r requirements.txt
```

### Run the service

```bash
FLASK_APP=service flask run --port 8000
# or
gunicorn --bind 0.0.0.0:8000 service:app
```

Verify it is running:

```bash
curl http://localhost:8000/health
curl -X POST http://localhost:8000/counters/foo
curl -X PUT  http://localhost:8000/counters/foo
```

### Lint

```bash
flake8 service --count --select=E9,F63,F7,F82 --show-source --statistics
flake8 service --count --max-complexity=10 --max-line-length=127 --statistics
```

### Test

```bash
nosetests -v --with-spec --spec-color --with-coverage --cover-package=app
```

### Build the container

```bash
docker build -t ci-cd-final-project:latest .
docker run --rm -p 8000:8000 ci-cd-final-project:latest
```

## Tekton / OpenShift delivery

```bash
# 1. Install the custom cleanup and nose tasks
kubectl apply -f .tekton/tasks.yml

# 2. Create the workspace PVC
kubectl apply -f .tekton/pvc.yml

# 3. Create the pipeline
kubectl apply -f .tekton/pipeline.yml

# 4. Run it
tkn pipeline start ci-cd-pipeline \
  --param git-url=https://github.com/<your-account>/ci-cd-final-project.git \
  --param app-name=ci-cd-final-project \
  --param build-image=image-registry.openshift-image-registry.svc:5000/<your-namespace>/ci-cd-final-project:latest \
  --workspace name=output,claimName=oc-lab-pvc \
  --showlog
```

See `SETUP_AND_DEPLOYMENT.md` for the full OpenShift setup, service account,
SCC, and registry details.

## Repository layout

```text
.github/workflows/workflow.yml   GitHub Actions CI workflow
.tekton/tasks.yml                Custom Tekton cleanup and nose tasks
.tekton/pipeline.yml             Tekton CI/CD pipeline
.tekton/pvc.yml                  PersistentVolumeClaim for the pipeline
service/                         Flask application package
tests/                           Unit tests
Dockerfile                       Container image definition
submission-evidence/             Captured verification evidence
AI_GRADING_ANSWERS.md            Grading question answers and evidence status
```

## License

Licensed under the Apache License. See [LICENSE](/LICENSE)

## Author

Skills Network

## <h3 align="center"> © IBM Corporation 2023. All rights reserved. <h3/>
