# AI Grading Answers

Project: **ci-cd-final-project**
GitHub account: `yaqyn`
Repository: <https://github.com/yaqyn/ci-cd-final-project>

Status legend:

- **VERIFIED** — generated and checked against the actual service.
- **PENDING** — requires OpenShift authentication/cluster access or a manual
  screenshot.
- **FAILED** — attempted but unsuccessful (reason given).

---

## Q1 — README GitHub URL (2 points) — VERIFIED

<https://github.com/yaqyn/ci-cd-final-project/blob/main/README.md>

The README contains the heading **CI/CD Tools and Practices Final Project
Template**, the project name `ci-cd-final-project`, a project description, a
CI/CD workflow overview, and setup/testing instructions.

---

## Q2 — GitHub Actions workflow URL (4 points) — VERIFIED

<https://github.com/yaqyn/ci-cd-final-project/blob/main/.github/workflows/workflow.yml>

The workflow is named `CI workflow`, triggers on pushes and pull requests to
`main`, runs on `ubuntu-latest` in the `python:3.9-slim` container, checks out
with `actions/checkout@v3`, installs dependencies, lints with flake8, and runs
nose.

Lint step:

```yaml
      - name: Lint with flake8
        run: |
          flake8 service --count --select=E9,F63,F7,F82 --show-source --statistics
          flake8 service --count --max-complexity=10 --max-line-length=127 --statistics
```

Unit test step:

```yaml
      - name: Run unit tests with nose
        run: |
          nosetests -v --with-spec --spec-color --with-coverage --cover-package=app
```

Dependency install step:

```yaml
      - name: Install dependencies
        run: |
          python -m pip install --upgrade pip
          pip install -r requirements.txt
```

Real check: the workflow was triggered on push and succeeded —
<https://github.com/yaqyn/ci-cd-final-project/actions/runs/42585417417>

---

## Q3 — Tekton tasks URL (4 points) — VERIFIED

<https://github.com/yaqyn/ci-cd-final-project/blob/main/.tekton/tasks.yml>

### Task A — cleanup

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: cleanup
spec:
  description: >-
    Cleans up the shared workspace by deleting all of its contents, including
    hidden files, while keeping the workspace directory itself.
  workspaces:
    - name: source
      description: The workspace to clean.
  steps:
    - name: remove
      image: alpine:3
      env:
        - name: WORKSPACE_SOURCE_PATH
          value: $(workspaces.source.path)
      workingDir: $(workspaces.source.path)
      securityContext:
        runAsNonRoot: false
        runAsUser: 0
      script: |
        #!/bin/sh
        set -eu

        TARGET="${WORKSPACE_SOURCE_PATH:-}"

        if [ -z "${TARGET}" ]; then
          echo "ERROR: WORKSPACE_SOURCE_PATH is empty; refusing to clean." >&2
          exit 1
        fi

        # Refuse obviously dangerous paths so a misconfigured workspace can
        # never wipe a system directory or the filesystem root.
        case "${TARGET}" in
          /|/bin|/boot|/dev|/etc|/home|/lib|/lib64|/opt|/proc|/root|/run|/sbin|/srv|/sys|/tmp|/usr|/var)
            echo "ERROR: refusing to clean protected path '${TARGET}'." >&2
            exit 1
            ;;
        esac

        if [ ! -d "${TARGET}" ]; then
          echo "ERROR: workspace '${TARGET}' is not a directory." >&2
          exit 1
        fi

        echo "Removing all files from ${TARGET} ..."
        cd "${TARGET}"
        # Remove regular and hidden entries, but never the directory itself.
        rm -rf -- ./* ./.[!.]* ./..?* 2>/dev/null || true
        echo "Workspace ${TARGET} is now empty."
```

### Task B — nose

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: nose
spec:
  description: Runs the nose unit test suite for the application.
  workspaces:
    - name: source
      description: The workspace containing the application source.
  params:
    - name: args
      description: Arguments to pass to nosetests.
      type: string
      default: "-v"
  steps:
    - name: nosetests
      image: python:3.9-slim
      workingDir: $(workspaces.source.path)
      script: |
        #!/bin/bash
        set -e
        python -m pip install --upgrade pip wheel
        pip install -r requirements.txt
        nosetests $(params.args)
```

Both tasks were applied to a real Tekton v1.6.0 API server and executed:

- `cleanup-run`: `Succeeded` — workspace had 15 entries including hidden files;
  after the run `count=0`; the workspace directory remained.
- `nose-run`: `Succeeded` — `Ran 9 tests ... OK`, 99% coverage.

Evidence: `submission-evidence/tekton-local-validation.txt`.

---

## Q4 — PVC screenshot (2 points) — PENDING

Required file: `oc-pipelines-console-pvc-details.png`

The PVC manifest is complete and API-validated:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: oc-lab-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: skills-network-learner
```

It could not be captured because this environment has **no OpenShift cluster,
`oc` CLI, or console access** (only a local minikube context). No screenshot was
fabricated.

Manual capture after cluster access:

1. Apply `.tekton/pvc.yml` and confirm `oc get pvc oc-lab-pvc` shows
   `Bound`.
2. Open the console: **Storage → PersistentVolumeClaims → oc-lab-pvc**.
3. Capture the panel showing name `oc-lab-pvc`, storage class
   `skills-network-learner`, capacity `1Gi`, and status `Bound`.
4. Save as `submission-evidence/oc-pipelines-console-pvc-details.png`.

---

## Q5 — GitHub Actions successful run output (2 points) — VERIFIED

Real terminal output captured with GitHub CLI:

```text
$ gh run list -R yaqyn/ci-cd-final-project --limit 5
completed	success	Add grading answers, setup guide, verification report, and evidence	CI workflow	main	push	42605673748	21s	2026-11-09T11:01:52Z
completed	success	Implement CI workflow, Tekton tasks, pipeline, and project docs	CI workflow	main	push	42585417417	23s	2026-11-09T08:22:01Z

$ gh run view 42585417417 -R yaqyn/ci-cd-final-project --verbose

✓ main CI workflow · 42585417417
Triggered via push

JOBS
✓ Build and test in 19s (ID 129053953845)
  ✓ Set up job
  ✓ Initialize containers
  ✓ Checkout
  ✓ Install dependencies
  ✓ Lint with flake8
  ✓ Run unit tests with nose
  ✓ Post Checkout
  ✓ Stop containers
  ✓ Complete job

$ gh run view 42605673748 -R yaqyn/ci-cd-final-project --verbose

✓ main CI workflow · 42605673748
Triggered via push

JOBS
✓ Build and test in 22s (ID 129120921448)
  ✓ Set up job
  ✓ Initialize containers
  ✓ Checkout
  ✓ Install dependencies
  ✓ Lint with flake8
  ✓ Run unit tests with nose
  ✓ Post Checkout
  ✓ Stop containers
  ✓ Complete job

View this run on GitHub: https://github.com/yaqyn/ci-cd-final-project/actions/runs/42605673748
```

Full output: `submission-evidence/cicd-github-validate.txt`.

Run URLs:
- <https://github.com/yaqyn/ci-cd-final-project/actions/runs/42585417417>
- <https://github.com/yaqyn/ci-cd-final-project/actions/runs/42605673748>

---

## Q6 — OpenShift pipeline screenshot (2 points) — PENDING

Required file: `oc-pipelines-oc-final.png`

The pipeline definition at `.tekton/pipeline.yml` contains, in dependency order:

1. `cleanup`
2. `git-clone`
3. `flake8`
4. `nose`
5. `buildah`
6. `deploy` (openshift-client)

The Pipeline resource was accepted by a real Tekton API server, but no
OpenShift console was available to capture the pipeline graph. No screenshot
was fabricated.

Manual capture:

1. `kubectl apply -f .tekton/pipeline.yml`
2. Open **Pipelines → Pipelines → ci-cd-pipeline → Pipeline details**.
3. Capture the graph/list showing all six tasks.
4. Save as `submission-evidence/oc-pipelines-oc-final.png`.

---

## Q7 — Successful pipeline screenshot (2 points) — PENDING

Required file: `oc-pipelines-oc-green.png`

No OpenShift cluster was reachable, so no pipeline run could be executed there.
The custom tasks were executed successfully on a local Tekton installation, but
that is **not** presented as an OpenShift pipeline run.

Manual capture after running the pipeline:

1. Run the pipeline using the command in `SETUP_AND_DEPLOYMENT.md` §3.8.
2. Open **Pipelines → Pipelines → ci-cd-pipeline → PipelineRuns → latest run**.
3. Capture the run with every task in Succeeded/green state.
4. Save as `submission-evidence/oc-pipelines-oc-green.png`.

---

## Q8 — OpenShift application logs (2 points) — PENDING

Required file: `submission-evidence/oc-pipelines-app-logs.txt`

Actual OpenShift application logs could not be captured because there is no
OpenShift deployment in this environment. The file
`submission-evidence/oc-pipelines-app-logs.txt` contains an honest PENDING
notice and the exact command to run; it does **not** contain invented log
entries.

For reference only (not a substitute for cluster logs), the application was run
locally in the built container and emitted the required banner:

```text
[2026-11-09 08:21:07 +0000] [1] [INFO] Starting gunicorn 20.1.0
[2026-11-09 08:21:07 +0000] [1] [INFO] Listening at: http://0.0.0.0:8000 (1)
[2026-11-09 08:21:07 +0000] [INFO] [log_handlers] Logging handler established
[2026-11-09 08:21:07 +0000] [INFO] [__init__] **********************************************************************
[2026-11-09 08:21:07 +0000] [INFO] [__init__] ******************  S E R V I C E   R U N N I N G  *******************
[2026-11-09 08:21:07 +0000] [INFO] [__init__] **********************************************************************
```

Local evidence: `submission-evidence/local-app-startup.log`.

Manual capture after deployment:

```bash
oc logs deployment/ci-cd-final-project | tee submission-evidence/oc-pipelines-app-logs.txt
```

The captured log must show `S E R V I C E   R U N N I N G`.

---

## Status summary

| Question | Points | Status |
| --- | --- | --- |
| Q1 — README URL | 2 | VERIFIED |
| Q2 — GitHub Actions workflow URL | 4 | VERIFIED |
| Q3 — Tekton tasks URL | 4 | VERIFIED |
| Q4 — PVC screenshot | 2 | PENDING |
| Q5 — GitHub Actions run output | 2 | VERIFIED |
| Q6 — OpenShift pipeline screenshot | 2 | PENDING |
| Q7 — Successful pipeline screenshot | 2 | PENDING |
| Q8 — OpenShift application logs | 2 | PENDING |

Verified points: 12/20. Remaining 8 points require OpenShift cluster access and
manual screenshots.
