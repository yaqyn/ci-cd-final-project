# Submission Evidence

| File | Description | Status |
| --- | --- | --- |
| `cicd-github-validate.txt` | Real `gh run list` / `gh run view --verbose` output for successful CI run 42585417417 | VERIFIED |
| `local-ci-run.txt` | flake8 + nose run inside `python:3.9-slim` (same as CI) | VERIFIED |
| `local-docker-build.txt` | `docker build` output for the application image | VERIFIED |
| `local-app-startup.log` | Gunicorn startup log including `S E R V I C E   R U N N I N G` | VERIFIED |
| `tekton-local-validation.txt` | Real Tekton v1.6.0 run of `cleanup` and `nose` tasks on minikube | VERIFIED |
| `oc-pipelines-app-logs.txt` | OpenShift application logs | PENDING — no cluster access |
| `oc-pipelines-console-pvc-details.png` | OpenShift PVC screenshot | PENDING — manual capture |
| `oc-pipelines-oc-final.png` | OpenShift pipeline configuration screenshot | PENDING — manual capture |
| `oc-pipelines-oc-green.png` | Successful OpenShift pipeline run screenshot | PENDING — manual capture |

No screenshot or log was simulated. See `AI_GRADING_ANSWERS.md` and
`SETUP_AND_DEPLOYMENT.md` for the exact commands and console navigation needed
to complete the pending evidence.
