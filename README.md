# Jenkins Pipelines — Zoop (Trivy + SonarQube)

This folder contains Jenkins pipeline files for 6 repos that don't have CI/CD set up yet.
Created by Adin — hand off to DevOps to deploy.

---

## Folder Structure

```
jenkins-pipelines/
├── zoop-customer-platform/
│   ├── dev/Jenkinsfile
│   ├── sandbox/Jenkinsfile
│   ├── stage/Jenkinsfile
│   └── prod/Jenkinsfile
├── stack/
├── face-liveliness/
├── face-liveliness-frontend/
├── digilocker/
└── zoop-digilocker-v1/
```

Each repo has 4 environment files. DevOps copies each `Jenkinsfile` into the matching repo under `jenkins/<env>/Jenkinsfile`.

---

## What Each Pipeline Does

### Stages (in order)

| Stage | What it does |
|---|---|
| **Clone & Setup** | Clones the app repo + `marine-ford` infra repo, merges `.env.secrets` + `.env.vars` |
| **SonarQube Analysis** | Runs code quality scan against `https://sonarqube.zoop.tools/` |
| **Trivy Filesystem Scan** | Scans source code for known CVEs (HIGH + CRITICAL) |
| **Trivy IaC Scan** | Scans Terraform/K8s/Dockerfile configs for misconfigurations |
| **Build** | Builds Docker image, pushes to Google Artifact Registry |
| **Deploy** | Deploys to GKE using `kubectl apply` |

> **Note:** SonarQube + Trivy scans only run when `RUN_TESTS = true` is checked at build time. This keeps normal deploys fast.

---

## Environment Matrix

| Env file | Agent | GCP Project | GKE Cluster | Namespace | Default Branch |
|---|---|---|---|---|---|
| `dev/Jenkinsfile` | dev-agent | zoop-one-development | development-k8s-cluster | develop | develop |
| `sandbox/Jenkinsfile` | prod-agent | zoop-production | zoop-one-production-cluster | sandbox | sandbox |
| `stage/Jenkinsfile` | dev-agent | zoop-one-development | development-k8s-cluster | staging | main |
| `prod/Jenkinsfile` | prod-agent | zoop-production | zoop-one-production-cluster | production | stable |

---

## SonarQube Project Keys

These are the project keys used in each Jenkinsfile. They must exist in SonarQube before the scan will work.

| Repo | SonarQube Project Key | Status |
|---|---|---|
| `zoop-customer-platform` | `zoop-customer-platform` | ❌ Needs to be created |
| `stack` | `stack` | ✅ Already exists |
| `face-liveliness` | `face-liveliness` | ❌ Needs to be created |
| `face-liveliness-frontend` | `face-liveliness-frontend` | ❌ Needs to be created |
| `digilocker` | `digilocker` | ❌ Needs to be created |
| `zoop-digilocker-v1` | `zoop-digilocker-v1` | ❌ Needs to be created |

### How to create a project in SonarQube (once you have admin access)
1. Go to `https://sonarqube.zoop.tools`
2. Click **Projects → Create Project → Manually**
3. Set **Project Key** exactly as shown in the table above
4. Set **Display Name** same as the key
5. Click **Set Up**
6. Go to **My Account → Security → Generate Tokens**
7. Generate one token, name it `jenkins-scanner`
8. Copy the token → give to DevOps → he adds it to Jenkins as a credential named `SONARQUBE_TOKEN`

---

## What DevOps Needs to Fill In

Search for `TODO` in any Jenkinsfile — there are 2 things per file:

### 1. `GAR_REPO` — Google Artifact Registry repo name
```
GAR_REPO = 'TODO_GAR_REPO'
```
Replace with the correct GAR repo for each service. Examples from existing pipelines:
- `dev-zoopsign` (dev)
- `uat-zoopsign` (stage)
- `prod-zoopsign` (sandbox + prod)

Ask DevOps what the equivalent repo names are for these 6 services.

### 2. `INFRA_FOLDER_NAME` and `GKE_DEPLOYMENT_NAME`
```
INFRA_FOLDER_NAME = 'REPO_NAME'  // must match folder in marine-ford/k8s/
GKE_DEPLOYMENT_NAME = 'REPO_NAME-deploy'  // must match K8s deployment name
```
DevOps needs to:
- Create the K8s deployment YAMLs in `marine-ford/k8s/<repo-name>/<env>/deployment.yaml`
- Confirm the deployment name matches what's in the YAML

---

## Trivy — Important Notes

### `--exit-code 0` (current setting)
The pipeline will **NOT fail** if vulnerabilities are found — it just reports them.
This is intentional so existing code doesn't block all deployments on day one.

### When to switch to `--exit-code 1`
Once DevOps + engineering have reviewed the first scan results and fixed critical issues, change to:
```groovy
sh "trivy fs --exit-code 1 --severity HIGH,CRITICAL ..."
```
This will make the pipeline **block** on HIGH/CRITICAL vulnerabilities — the right long-term behavior.

### Trivy must be installed on Jenkins agents
DevOps needs to confirm Trivy is installed on both `dev-agent` and `prod-agent`.
Install command (Ubuntu): `apt-get install trivy` or via the Trivy install script.

---

## How to Deploy These Files

DevOps steps per repo:

1. Clone the target repo (e.g. `git clone https://github.com/zoop/digilocker`)
2. Create the `jenkins/` folder: `mkdir -p jenkins/{dev,sandbox,stage,prod}`
3. Copy the Jenkinsfiles from this folder into the repo
4. Commit and push: `git add jenkins/ && git commit -m "add jenkins pipeline" && git push`
5. In Jenkins: create a new Pipeline job, point it to the repo + `jenkins/<env>/Jenkinsfile`
6. Run once with `RUN_TESTS = false` to verify build + deploy works
7. Then run with `RUN_TESTS = true` to test SonarQube + Trivy scans

---

## Existing Repos Already on Jenkins (for reference)

These repos already have working pipelines — use them as reference:
- `zoop/kaido-payment-sdk` → `jenkins/dev/Jenkinsfile`
- `zoop/zsp-admin-portal` → `jenkins/dev/Jenkinsfile`
- `zoop/zou` → `jenkins/dev/Jenkinsfile`

> Those pipelines currently use `snyk test` instead of Trivy. DevOps can update those too by replacing the Security Scan stage with the Trivy stages from this folder.

---

## Questions? Contact

- Raise with DevOps team
- SonarQube admin access: Adin is requesting it (pending as of July 2026)
- SonarQube URL: `https://sonarqube.zoop.tools`
