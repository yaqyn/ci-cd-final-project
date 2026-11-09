# Setup and Deployment Guide

Project: **ci-cd-final-project** — Flask Hit Counter Service
Repository: <https://github.com/yaqyn/ci-cd-final-project>

This guide covers local development, the GitHub Actions CI workflow, and the
Tekton/OpenShift continuous delivery pipeline.

---

## 1. Local development

### Prerequisites

- Python 3.9 (required by the CI container and the Dockerfile)
- Git
- Docker (optional, for the container build)

### Setup

```bash
git clone https://github.com/yaqyn/ci-cd-final-project.git
cd ci-cd-final-project

python3.9 -m venv venv
source venv/bin/activate
python -m pip install --upgrade pip wheel
pip install -r requirements.txt
```

### Run

```bash
gunicorn --bind 0.0.0.0:8000 service:app
# or
FLASK_APP=service flask run --port 8000
```

```bash
curl http://localhost:8000/health
# {"status":"OK"}

curl -X POST http://localhost:8000/counters/foo
curl -X PUT  http://localhost:8000/counters/foo
curl http://localhost:8000/counters/foo
```

Startup logs include the banner:

```text
******************  S E R V I C E   R U N N I N G  *******************
```

### Lint and test

```bash
flake8 service --count --select=E9,F63,F7,F82 --show-source --statistics
flake8 service --count --max-complexity=10 --max-line-length=127 --statistics

nosetests -v --with-spec --spec-color --with-coverage --cover-package=app
```

Expected result: `Ran 9 tests ... OK` and 99% coverage of the `service`
package. The `--cover-package=app` argument is the lab-required command; since
the package is named `service`, coverage emits a harmless
`Module app was never imported` warning while still reporting on `service`.

---

## 2. GitHub Actions CI

Workflow: `.github/workflows/workflow.yml`

- Name: `CI workflow`
- Triggers: `push` to `main`, `pull_request` targeting `main`
- Runner: `ubuntu-latest`
- Container: `python:3.9-slim`
- Steps: checkout (`actions/checkout@v3`), dependency install, flake8,
  nose unit tests.

Push to `main` and the workflow runs automatically. Verify with:

```bash
gh run list -R yaqyn/ci-cd-final-project
gh run view <run-id> -R yaqyn/ci-cd-final-project --verbose
```

Actual successful run: <https://github.com/yaqyn/ci-cd-final-project/actions/runs/42585417417>

---

## 3. OpenShift / Tekton continuous delivery

> **Environment note:** this workspace had GitHub access but **no OpenShift
> cluster, no `oc` CLI, and no `tkn` CLI**. The manifests below are complete and
> were validated against a real Tekton v1.6.0 installation (the `cleanup` and
> `nose` tasks were executed successfully). The full pipeline, PVC, image
> build, and OpenShift deployment require an authenticated OpenShift cluster.

### 3.1 Prerequisites on OpenShift

- OpenShift 4.x with the **OpenShift Pipelines** operator installed
- Logged in with `oc login ...`
- A project/namespace, for example:

```bash
oc new-project ci-cd-final-project
```

### 3.2 Install the custom tasks

```bash
kubectl apply -f .tekton/tasks.yml
# or
oc apply -f .tekton/tasks.yml
```

This creates the `cleanup` and `nose` tasks in the current namespace.

### 3.3 Install the catalog tasks used by the pipeline

The pipeline references the standard Tekton Catalog tasks `git-clone`,
`flake8`, `buildah`, and `openshift-client`. Install them with Tekton Hub:

```bash
tkn hub install task git-clone
tkn hub install task flake8
tkn hub install task buildah
tkn hub install task openshift-client
```

If `tkn hub` is unavailable, install the YAML from the Tekton Catalog:

- <https://github.com/tektoncd/catalog/tree/main/task/git-clone>
- <https://github.com/tektoncd/catalog/tree/main/task/flake8>
- <https://github.com/tektoncd/catalog/tree/main/task/buildah>
- <https://github.com/tektoncd/catalog/tree/main/task/openshift-client>

### 3.4 Create the workspace PVC

```bash
kubectl apply -f .tekton/pvc.yml
kubectl get pvc oc-lab-pvc
```

The PVC is named `oc-lab-pvc`, requests `1Gi`, and uses the lab storage class
`skills-network-learner` (as required by the course lab).

### 3.5 Create the pipeline

```bash
kubectl apply -f .tekton/pipeline.yml
kubectl get pipeline ci-cd-pipeline
```

Pipeline task order and dependencies:

```text
cleanup -> git-clone -> flake8 -> nose -> buildah -> deploy
```

### 3.6 Service account and permissions

```bash
oc create serviceaccount pipeline 2>/dev/null || true

# Buildah needs privileged access to build images.
oc adm policy add-scc-to-user privileged -z pipeline

# The deploy step creates/applies Deployments and Services.
oc adm policy add-role-to-user edit -z pipeline

# Allow the pipeline service account to push to the internal registry.
oc policy add-role-to-user system:image-builder -z pipeline

# Use the SA for the pipeline run.
oc secrets link pipeline $(oc get sa pipeline -o jsonpath='{.secrets[0].name}') 2>/dev/null || true
```

### 3.7 Determine the internal registry image reference

```bash
# Namespace and registry host used by the OpenShift internal registry
NAMESPACE=$(oc project -q)
REGISTRY=$(oc get route default-route -n openshift-image-registry \
  -o jsonpath='{.spec.host}' 2>/dev/null)

echo "Internal registry service: image-registry.openshift-image-registry.svc:5000"
echo "External route:            ${REGISTRY}"
```

Using the internal service DNS name (recommended inside the cluster):

```text
image-registry.openshift-image-registry.svc:5000/<namespace>/ci-cd-final-project:latest
```

### 3.8 Run the pipeline

```bash
tkn pipeline start ci-cd-pipeline \
  --serviceaccount pipeline \
  --param git-url=https://github.com/yaqyn/ci-cd-final-project.git \
  --param git-revision=main \
  --param app-name=ci-cd-final-project \
  --param build-image=image-registry.openshift-image-registry.svc:5000/$(oc project -q)/ci-cd-final-project:latest \
  --workspace name=output,claimName=oc-lab-pvc \
  --showlog
```

Or create a `PipelineRun`:

```bash
cat <<EOF | kubectl apply -f -
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: ci-cd-pipeline-run-
spec:
  pipelineRef:
    name: ci-cd-pipeline
  serviceAccountName: pipeline
  params:
    - name: git-url
      value: https://github.com/yaqyn/ci-cd-final-project.git
    - name: git-revision
      value: main
    - name: app-name
      value: ci-cd-final-project
    - name: build-image
      value: image-registry.openshift-image-registry.svc:5000/$(oc project -q)/ci-cd-final-project:latest
  workspaces:
    - name: output
      persistentVolumeClaim:
        claimName: oc-lab-pvc
EOF
```

Watch it:

```bash
tkn pipelinerun logs -f -L
# or
kubectl get pipelinerun -w
```

### 3.9 Verify the deployment

```bash
oc get deployment ci-cd-final-project
oc get pods -l app=ci-cd-final-project
oc get service ci-cd-final-project
oc get route ci-cd-final-project

oc rollout status deployment/ci-cd-final-project
oc logs deployment/ci-cd-final-project
```

The deploy task is idempotent and uses the lab-required command:

```bash
oc create deployment $(params.app-name) \
  --image=$(params.build-image) \
  --dry-run=client -o yaml | oc apply -f -
```

It then exposes the Deployment on port 8000 if no Service exists, and waits
for the rollout to complete.

---

## 4. Manual evidence capture (OpenShift console)

OpenShift access was not available in this environment, so the following
screenshots/logs must be captured manually after running the pipeline:

### Evidence A — PVC details

Console: **Storage → PersistentVolumeClaims → oc-lab-pvc**

Capture the panel showing name `oc-lab-pvc`, storage class
`skills-network-learner`, capacity `1Gi`, and status `Bound`.
Save as `oc-pipelines-console-pvc-details.png`.

CLI equivalent:

```bash
oc get pvc oc-lab-pvc -o wide
oc describe pvc oc-lab-pvc
```

### Evidence B — Pipeline configuration

Console: **Pipelines → Pipelines → ci-cd-pipeline → Pipeline details**

Capture the graph/list showing cleanup, git-clone, flake8, nose, buildah, and
deploy.
Save as `oc-pipelines-oc-final.png`.

```bash
tkn pipeline describe ci-cd-pipeline
```

### Evidence C — Successful pipeline run

Console: **Pipelines → Pipelines → ci-cd-pipeline → PipelineRuns → latest run**

Capture all tasks with green/succeeded status.
Save as `oc-pipelines-oc-green.png`.

```bash
tkn pipelinerun describe <run-name>
kubectl get pipelinerun
```

### Evidence D — Application logs

```bash
oc logs deployment/ci-cd-final-project | tee submission-evidence/oc-pipelines-app-logs.txt
```

The log must show `S E R V I C E   R U N N I N G`.

---

## 5. Repository files

| Path | Purpose |
| --- | --- |
| `.github/workflows/workflow.yml` | GitHub Actions CI |
| `.tekton/tasks.yml` | `cleanup` and `nose` Tekton tasks |
| `.tekton/pipeline.yml` | Full CI/CD pipeline |
| `.tekton/pvc.yml` | `oc-lab-pvc` PersistentVolumeClaim |
| `service/` | Flask application |
| `tests/` | Unit tests |
| `Dockerfile` | Container image |
| `submission-evidence/` | Verification evidence |
