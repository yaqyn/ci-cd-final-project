# Verification Report

Project: **ci-cd-final-project**
Repository: <https://github.com/yaqyn/ci-cd-final-project>
Verification date: 2026-11-09 (UTC)

## Environment inspected

| Tool / resource | Result |
| --- | --- |
| Git | 2.56.0 |
| GitHub CLI | authenticated as `yaqyn` (token redacted), scopes include `repo`, `workflow` |
| Python (host) | 3.15.2 — not suitable for Flask 2.1.2 / nose 1.3.7; tests were run in Python 3.9 |
| Docker | 29.9.2 (server) |
| `oc` CLI | **not installed** |
| `tkn` CLI | **not installed** |
| OpenShift cluster / kube context | **not available** — only a local `minikube` context exists |
| `kubectl` | 1.37.1 |
| minikube | 1.39.0, Kubernetes 1.35.1, Docker driver |

Because no OpenShift cluster, `oc`, or `tkn` was available, OpenShift-only
evidence (PVC screenshot, pipeline screenshots, application logs from the
cluster) could not be captured. It is marked **PENDING** in
`AI_GRADING_ANSWERS.md`; nothing was simulated.

## 1. Repository and files

Commands:

```bash
git clone https://github.com/ibm-developer-skills-network/vselh-ci-cd-final-project-template.git ci-cd-final-project
gh repo create ci-cd-final-project --public --source=. --remote=origin --push
git push origin main
```

Result: **PASS** — public repository created at
<https://github.com/yaqyn/ci-cd-final-project>, template application and tests
preserved.

Implemented files:

- `README.md`
- `.github/workflows/workflow.yml`
- `.tekton/tasks.yml`
- `.tekton/pipeline.yml`
- `.tekton/pvc.yml`
- `SETUP_AND_DEPLOYMENT.md`
- `AI_GRADING_ANSWERS.md`
- `VERIFICATION_REPORT.md`
- `submission-evidence/` (evidence)

## 2. YAML validation

Command:

```bash
python -c "import yaml; [list(yaml.safe_load_all(open(f))) for f in [...]]"
```

Result: **PASS**

```text
OK .github/workflows/workflow.yml -> 1 document(s)
OK .tekton/tasks.yml -> 2 document(s)
OK .tekton/pipeline.yml -> 1 document(s)
OK .tekton/pvc.yml -> 1 document(s)
```

Tekton server-side validation (real Tekton API server, v1.6.0):

```bash
kubectl apply -f .tekton/tasks.yml
kubectl apply --dry-run=server -f .tekton/pipeline.yml
kubectl apply --dry-run=server -f .tekton/pvc.yml
kubectl apply -f .tekton/pipeline.yml
```

Result: **PASS**

```text
task.tekton.dev/cleanup created
task.tekton.dev/nose created
pipeline.tekton.dev/ci-cd-pipeline created (server dry run)
persistentvolumeclaim/oc-lab-pvc created (server dry run)
pipeline.tekton.dev/ci-cd-pipeline created
```

Note: `pvc.yml` uses the lab-specific storage class
`skills-network-learner`; it validated against the API server but was not bound
because the OpenShift storage class is not present on minikube.

## 3. Lint (flake8)

Commands (run in `python:3.9-slim`, matching CI):

```bash
flake8 service --count --select=E9,F63,F7,F82 --show-source --statistics
flake8 service --count --max-complexity=10 --max-line-length=127 --statistics
```

Result: **PASS** — both commands exited 0 with `0` reported violations.

## 4. Unit tests (nose)

Command:

```bash
nosetests -v --with-spec --spec-color --with-coverage --cover-package=app
```

Result: **PASS**

```text
Ran 9 tests in 0.241s

OK
TOTAL 112 statements, 1 miss, 99% coverage
```

The `Module app was never imported` warning is expected because the required
command uses `--cover-package=app` while the package is named `service`; the
tests still pass and coverage is reported for `service`.

Full output: `submission-evidence/local-ci-run.txt`.

## 5. Container build and application runtime

Commands:

```bash
docker build -t ci-cd-final-project:local .
docker run -d --name cicd-app-test -p 18000:8000 ci-cd-final-project:local
curl -sS -i http://127.0.0.1:18000/health
curl -sS -X POST http://127.0.0.1:18000/counters/foo
curl -sS -X PUT  http://127.0.0.1:18000/counters/foo
docker logs cicd-app-test
```

Result: **PASS**

```text
HTTP/1.1 200 OK
{"status":"OK"}
{"counter":0,"name":"foo"}
{"counter":1,"name":"foo"}
...
[INFO] [__init__] ******************  S E R V I C E   R U N N I N G  *******************
```

Evidence: `submission-evidence/local-app-startup.log`,
`submission-evidence/local-docker-build.txt`.

## 6. GitHub Actions

Command:

```bash
gh run list -R yaqyn/ci-cd-final-project
gh run view 42585417417 -R yaqyn/ci-cd-final-project --verbose
```

Result: **PASS**

```text
✓ main CI workflow · 42585417417
✓ Build and test in 19s
  ✓ Set up job
  ✓ Initialize containers
  ✓ Checkout
  ✓ Install dependencies
  ✓ Lint with flake8
  ✓ Run unit tests with nose
  ✓ Post Checkout
  ✓ Stop containers
  ✓ Complete job
```

Second successful run after the documentation commit:

```text
✓ main CI workflow · 42605673748
✓ Build and test in 22s
  ✓ Checkout / Install dependencies / Lint with flake8 / Run unit tests with nose
```

Run URLs:
- <https://github.com/yaqyn/ci-cd-final-project/actions/runs/42585417417>
- <https://github.com/yaqyn/ci-cd-final-project/actions/runs/42605673748>

Raw output: `submission-evidence/cicd-github-validate.txt`.

## 7. Tekton task functional validation

A real Tekton Pipelines v1.6.0 installation was created on the local
`minikube` cluster. The custom tasks were applied and executed against a 1Gi
PVC workspace containing the repository.

Commands:

```bash
kubectl apply -f .tekton/tasks.yml
kubectl apply -f .tekton/pipeline.yml
kubectl create -f <PVC manifest>          # minikube-compatible local PVC
kubectl cp . workspace-seed:/workspace
kubectl create -f <cleanup TaskRun>
kubectl create -f <nose TaskRun>
kubectl logs cleanup-run-pod -c step-remove
kubectl logs nose-run-pod -c step-nosetests
```

Results:

- `cleanup`: **PASS** — `TaskRun cleanup-run` = `True / Succeeded`; workspace
  contained 15 entries including hidden files before the run and `count=0`
  afterwards (verified from a separate pod). The workspace directory itself was
  preserved.
- `nose`: **PASS** — `TaskRun nose-run` = `True / Succeeded`; 9 tests passed,
  99% coverage.
- `pipeline`: **PASS (definition only)** — accepted by the Tekton API server;
  not executed because it requires catalog tasks `git-clone`, `flake8`,
  `buildah`, `openshift-client` plus an OpenShift registry and cluster.

Evidence: `submission-evidence/tekton-local-validation.txt`.

Local validation workarounds (not part of the repository): the minikube CoreDNS
forwarder was pointed at a reachable DNS resolver, and the Tekton controller's
shell image was changed to a locally available image so tasks could run without
external image pulls.

## 8. OpenShift pipeline, PVC, deployment, and logs

Result: **NOT RUN — BLOCKED**

Reason: no OpenShift cluster, no `oc`, no `tkn`, and no authenticated OpenShift
context in this environment. The only Kubernetes context is a local minikube
cluster. No OpenShift evidence was fabricated.

Required manual steps after gaining cluster access:

```bash
# install tasks / catalog tasks / PVC / pipeline
kubectl apply -f .tekton/tasks.yml
tkn hub install task git-clone
tkn hub install task flake8
tkn hub install task buildah
tkn hub install task openshift-client
kubectl apply -f .tekton/pvc.yml
kubectl apply -f .tekton/pipeline.yml

# permissions
oc adm policy add-scc-to-user privileged -z pipeline
oc adm policy add-role-to-user edit -z pipeline
oc policy add-role-to-user system:image-builder -z pipeline

# run
tkn pipeline start ci-cd-pipeline \
  --serviceaccount pipeline \
  --param git-url=https://github.com/yaqyn/ci-cd-final-project.git \
  --param app-name=ci-cd-final-project \
  --param build-image=image-registry.openshift-image-registry.svc:5000/$(oc project -q)/ci-cd-final-project:latest \
  --workspace name=output,claimName=oc-lab-pvc \
  --showlog

# evidence
oc get pvc oc-lab-pvc -o wide
tkn pipeline describe ci-cd-pipeline
kubectl get pipelinerun
oc logs deployment/ci-cd-final-project | tee submission-evidence/oc-pipelines-app-logs.txt
```

Then capture the three console screenshots listed in
`SETUP_AND_DEPLOYMENT.md` section 4.

## 9. Repository hygiene

- The template application (`service/`) and tests (`tests/`) are unchanged.
- No credentials or tokens were written to any file or printed.
- The unrelated remote `upstream` points to the IBM template; `origin` points to
  the submission repository.
- Two automatically-created Dependabot pull requests failed CI because their
  dependency bumps are incompatible with the pinned Flask/Werkzeug versions;
  the open one was closed so it does not affect the submission branch. The
  `main` branch CI run is green.

## Unresolved issues / required manual actions

1. **OpenShift access** is required for Q4, Q6, Q7, and Q8 (PVC screenshot,
   pipeline configuration screenshot, successful-run screenshot, application
   logs from the cluster).
2. **Screenshots** must be captured from the OpenShift web console; no browser
   automation against an OpenShift console was available.
3. The full Tekton pipeline has not been executed end-to-end; its tasks were
   validated individually and the Pipeline definition was accepted by a real
   Tekton API server.

## Summary

| Area | Status |
| --- | --- |
| Repo + README | PASS |
| GitHub Actions workflow | PASS (green run 42585417417) |
| Tekton `cleanup` task | PASS (executed on Tekton v1.6.0) |
| Tekton `nose` task | PASS (executed on Tekton v1.6.0) |
| Tekton pipeline definition | PASS (server-validated, not end-to-end) |
| PVC manifest | VALID (not bound; OpenShift storage class absent) |
| OpenShift deployment + logs | PENDING (no cluster access) |
| Screenshots | PENDING (manual capture required) |
