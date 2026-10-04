# Car Rental Portal — CI/CD with GitHub Actions

## 🚀 A DevOps Project, Built One Level at a Time

This isn't a project that showed up finished. It's being **leveled up stage by stage**, on purpose, the same way most real teams actually adopt DevOps: start with something that just runs, then containerize it, then orchestrate it, then automate it, then take it to the cloud. Each stage is a complete, working checkpoint before the next one begins.

If you're a student following along, that's the point: **don't skip to the end**. Clone each stage, see what changed and why, and build the muscle memory one layer at a time instead of inheriting a finished black box.

```
  Stage 1          Stage 2                Stage 3                      Stage 4
┌─────────┐      ┌───────────┐        ┌──────────────────┐        ┌────────────────┐
│ Plain   │ ───► │ Dockerized │ ───►  │  + Kubernetes      │ ───► │  + AWS (EKS /   │
│ PHP app │      │  (app runs │       │  (self-healing,    │      │  ECR / RDS)     │
│         │      │  in a      │       │  scalable pods)    │      │  production-    │
│         │      │  container)│       │       +             │      │  grade cloud    │
│         │      │            │       │   CI/CD (this repo)│      │  deployment     │
│         │      │            │       │  (build→push→      │      │                 │
│         │      │            │       │   deploy, hands-off)│      │                 │
└─────────┘      └───────────┘        └──────────────────┘        └────────────────┘
```

| Stage | What it covers | Where |
|---|---|---|
| 1. Dockerize | Take a plain PHP app and containerize it | [`Carrental_Docker_Project`](https://github.com/chetan080808/Carrental_Docker_Project) |
| 2. Docker + Kubernetes | Deploy the containers on a Kubernetes cluster | [`Carrental_K8s-Docker_Project`](https://github.com/chetan080808/Carrental_K8s-Docker_Project) |
| 3. **Docker + K8s + CI/CD** | **Automate build → push → deploy with GitHub Actions** | **👉 this repo** |
| 4. AWS Migration | Move the whole stack to EKS / ECR / RDS | upcoming |

This README focuses on stage 3 — the pipeline that turns "I manually build, push, and `kubectl apply` every time" into "I push code and the rest happens by itself." For how the app was containerized and how the Kubernetes manifests in `k8s/` work in detail, see the [Docker + Kubernetes repo](https://github.com/chetan080808/Carrental_K8s-Docker_Project) that's required reading before this stage makes sense.

---

## Project Structure

```
car-rental-k8s/
├── .github/
│   └── workflows/
│       └── ci-cd.yml           # The GitHub Actions pipeline — see below
├── .gitignore                  # Excludes .claude/ and OS/editor junk
├── app/                        # PHP web application (Dockerfile builds this)
├── mysql/                      # MySQL Dockerfile + schema dump
├── k8s/                        # Kubernetes manifests, applied by the deploy job
└── README.md
```

---

## Prerequisites

Before touching the pipeline, you need stages 1–2 already done:

| Requirement | Why |
|---|---|
| A Kubernetes cluster already running (Minikube/k3s/kind) | The `deploy` job runs `kubectl` against it — build this first using the [Docker + Kubernetes repo](https://github.com/chetan080808/Carrental_K8s-Docker_Project) |
| An EC2 instance hosting that cluster | This is where the self-hosted runner will live |
| `kubectl` on that EC2 instance, already pointed at the cluster | Verify with `kubectl get nodes` before continuing |
| A Docker Hub account | Destination registry for built images |
| Admin access to this GitHub repo's **Settings** | To add secrets and register the runner |

If any of the above isn't done yet, do that first, this README assumes it's working.

---

## CI/CD with GitHub Actions

This is the core of the repo: `.github/workflows/ci-cd.yml`.

### Pipeline Overview

```
┌─────────────────────┐
│ PR → main            │──► lint-and-validate   (only this job runs)
└─────────────────────┘

┌─────────────────────┐     ┌──────────────────┐     ┌────────────────────────┐
│ push → main          │────►│ lint-and-validate │────►│ build-and-push         │
│ (merge / direct push)│     │ (GitHub-hosted)   │     │ (GitHub-hosted)        │
└─────────────────────┘     └──────────────────┘     └───────────┬────────────┘
                                                                   │
                                                                   ▼
                                                    ┌───────────────────────────┐
                                                    │ deploy                    │
                                                    │ (self-hosted runner, EC2) │
                                                    │ kubectl apply + set image │
                                                    └───────────────────────────┘
```

A pull request only runs lint/validate, nothing is built, pushed, or deployed. Only a push (or merge) to `main` runs the full chain.

### Job-by-Job Breakdown

#### 1. `lint-and-validate` - runs on every PR and push to `main`
| Step | What it does |
|---|---|
| Checkout | Clones the repo |
| Set up PHP 8.2 | Matches the version in `app/Dockerfile` |
| PHP syntax check | Runs `php -l` on every `.php` file under `app/` — catches syntax errors before they ship |
| Install kubeconform | Downloads the `kubeconform` binary |
| Validate K8s manifests | Runs `kubeconform -strict -summary -ignore-missing-schemas k8s/*.yaml` — catches malformed YAML or invalid Kubernetes fields |

If either check fails, the pipeline stops here — nothing downstream runs.

#### 2. `build-and-push` — only on push to `main`, after lint passes
| Step | What it does |
|---|---|
| Checkout | Clones the repo |
| Compute image tag | Derives `sha-<first 7 chars of commit SHA>` so every build is traceable to a commit |
| Set up Docker Buildx | Enables layer caching between runs |
| Log in to Docker Hub | Using the `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` secrets |
| Build & push web image | `app/Dockerfile` → `  <user>/carrental-web:latest` and `:sha-xxxxxxx` |
| Build & push mysql image | `mysql/Dockerfile` → `<user>/carrental-mysql:latest` and `:sha-xxxxxxx` |

The `sha-xxxxxxx` tag is passed to the `deploy` job as an output, so deploy always rolls out the *exact* image that was just built — never a stale `:latest`.

#### 3. `deploy` — only after `build-and-push` succeeds, runs on your self-hosted runner
| Step | What it does |
|---|---|
| Checkout | Pulls the latest `k8s/` manifests |
| Verify kubectl access | `kubectl config current-context && kubectl get nodes` — fails fast with a clear error if the runner can't reach the cluster |
| Apply manifests | `kubectl apply -f k8s/` — picks up any manifest changes (new ConfigMap keys, resource limits, etc.) |
| Roll out new web image | `kubectl set image deployment/carrental-web carrental-web=<user>/carrental-web:sha-xxxxxxx -n carrental`, then `kubectl rollout status` waits up to 180s for the new pods to become ready |

This job is gated behind a GitHub **Environment** named `production` (see Step 6 below) so you can require manual approval before it touches the cluster.

> `k8s/04-mysql-deployment.yaml` runs stock `mysql:8.0`, not the custom `mysql/` image — so `deploy` only rolls out `carrental-web`. The mysql image is still built/pushed for parity with local `docker-compose`.

---

## Setting Up the Pipeline — Step by Step

### Step 1 — Create a Docker Hub Access Token
1. Log in to [hub.docker.com](https://hub.docker.com)
2. **Account Settings → Security → New Access Token**
3. Name it (e.g. `github-actions-carrental`), copy the token — you won't see it again

### Step 2 — Add Repository Secrets
In this GitHub repo: **Settings → Secrets and variables → Actions → New repository secret**

| Secret name | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | The access token from Step 1 |

### Step 3 — Confirm the EC2 Instance Is Ready
SSH into the EC2 box that hosts your Kubernetes cluster and confirm `kubectl` already works:
```bash
kubectl get nodes
kubectl get pods -n carrental
```
If this doesn't work yet, go set up the cluster first (see the [Docker + Kubernetes repo](https://github.com/chetan080808/Carrental_K8s-Docker_Project)) — the runner itself doesn't install or configure Kubernetes, it just needs `kubectl` to already be pointed at a working cluster.

### Step 4 — Get a Runner Registration Token
In this GitHub repo: **Settings → Actions → Runners → New self-hosted runner → Linux → x64**

GitHub shows a `./config.sh --url ... --token ...` command with a short-lived token — copy it (or just the token), you'll use it in the next step.

### Step 5 — Install the Runner on EC2
Run these on the EC2 instance, as a dedicated non-root user if possible:

```bash
# 1. Create a folder and download the runner
mkdir actions-runner && cd actions-runner
curl -o actions-runner.tar.gz -L \
  https://github.com/actions/runner/releases/latest/download/actions-runner-linux-x64.tar.gz
tar xzf actions-runner.tar.gz

# 2. Register it against this repo (use the URL/token from Step 4)
./config.sh --url https://github.com/chetan080808/Carrental_Ci-CD --token <TOKEN_FROM_STEP_4>
# Accept the default name and the default label "self-hosted" when prompted —
# the workflow's `runs-on: [self-hosted]` targets that label directly.

# 3. Install and start it as a systemd service, so it survives reboots
sudo ./svc.sh install
sudo ./svc.sh start
```

Notes:
- The runner only makes **outbound** HTTPS (443) connections to GitHub to poll for jobs — no inbound security group rules are needed for it.
- It must run as (or have access to) whichever OS user has a working `kubectl` config (`~/.kube/config`) for the cluster — check this matches the user `svc.sh` runs the service as.

### Step 6 — Verify the Runner Is Online
- **GitHub UI**: Settings → Actions → Runners should show it with a green "Idle" dot.
- **On the EC2 box**: `sudo ./svc.sh status` should show it active/running.

### Step 7 — (Recommended) Add a Manual Approval Gate
**Settings → Environments → New environment → `production`**, then add yourself as a **required reviewer**.

The `deploy` job already declares `environment: production`, so once this exists, every run pauses for your approval before `kubectl` touches the cluster — a safety net against an accidental bad merge auto-deploying.

---

## Running & Watching the Pipeline

- **Trigger it**: push or merge a commit to `main` (or open a PR to see just lint/validate run).
- **Watch it**: GitHub repo → **Actions** tab → click the running workflow to see live logs per job.
- **Self-hosted job logs**: also written locally on the EC2 box under `actions-runner/_diag/`.
- **Re-run a failed job**: from the workflow run page → "Re-run jobs" → "Re-run failed jobs" (keeps the same image tag, doesn't rebuild if only `deploy` failed... actually it does rebuild since build-and-push also re-runs — that's expected and harmless, just re-pushes the same commit's image).

### Rolling Back a Bad Deploy
```bash
# See rollout history
kubectl rollout history deployment/carrental-web -n carrental

# Roll back to the previous version
kubectl rollout undo deployment/carrental-web -n carrental

# Or pin to a specific known-good commit's image
kubectl set image deployment/carrental-web \
  carrental-web=<user>/carrental-web:sha-<good-commit> -n carrental
```

---

## Troubleshooting the Pipeline

| Symptom | Likely cause / fix |
|---|---|
| Runner shows offline in GitHub UI | `sudo ./svc.sh status` on EC2 — restart with `sudo ./svc.sh start`; check outbound 443 to `github.com`/`*.actions.githubusercontent.com` isn't blocked by the security group or a NACL |
| `docker/login-action` step fails | `DOCKERHUB_TOKEN` expired or revoked — generate a new one (Step 1) and update the repo secret |
| `deploy` fails at "Verify kubectl access" | The OS user running the runner service doesn't have a valid `~/.kube/config` — confirm with `kubectl get nodes` as that same user |
| `kubectl rollout status` times out | New pod isn't becoming Ready — check `kubectl describe pod -n carrental <pod>` and `kubectl logs` for probe failures or `ImagePullBackOff` (image tag mismatch or Docker Hub auth issue) |
| `deploy` job never starts | It only runs after `build-and-push`, which only runs on a **push to `main`** — PRs intentionally skip it |
| Workflow doesn't trigger at all | Confirm the push actually landed on `main` and `.github/workflows/ci-cd.yml` is on that branch |

---

## Decommissioning the Runner

If you ever need to remove the runner (e.g. replacing the EC2 instance):
```bash
sudo ./svc.sh stop
sudo ./svc.sh uninstall
./config.sh remove --token <removal-token-from-GitHub-UI>
```

---

## Accessing the Deployed App

Once `deploy` succeeds, the app is reachable the same way it was in the Docker+K8s stage — NodePort or Ingress, depending on how `k8s/07-web-service.yaml` / `k8s/10-ingress.yaml` are configured. Full access instructions (node IP, ports, Ingress hosts file setup) are in the [Docker + Kubernetes repo](https://github.com/chetan080808/Carrental_K8s-Docker_Project).

**Default credentials** (seeded by `carrental.sql`):

| Area | Username | Password |
|---|---|---|
| Admin Panel (`/admin`) | `admin` | `Test@123` |
| phpMyAdmin / MySQL root | `root` | `rootpassword` |

---

## Roadmap — Stage 4: AWS Migration

Once this pipeline is solid, the next stage replaces the self-managed EC2+Minikube cluster with:
- **EKS** instead of Minikube/k3s on EC2
- **ECR** instead of (or alongside) Docker Hub
- **RDS** instead of the in-cluster MySQL pod
- A GitHub-hosted (not self-hosted) deploy job authenticating to AWS via OIDC, since EKS is reachable over the internet

Also worth exploring regardless of AWS timing:
- **Helm charts** to replace the raw `k8s/*.yaml` files
- **Horizontal Pod Autoscaler (HPA)**
- **Sealed Secrets / Vault** instead of base64 Secrets
