# Car Rental Portal — Kubernetes Deployment

A PHP + MySQL car rental web application deployed on Kubernetes.  
This project is designed as a hands-on learning guide for students to understand
containerisation, Docker image building, and Kubernetes fundamentals.

---

## Architecture

```
                        ┌─────────────────────────────────────────┐
                        │           Kubernetes Cluster             │
                        │         Namespace: carrental             │
                        │                                          │
  Browser               │  ┌──────────────┐   ┌────────────────┐  │
    │                   │  │  carrental-  │   │  phpmyadmin    │  │
    │  :30080 (web)     │  │  web (x2)    │   │  (x1)          │  │
    ├──────────────────►│  │  PHP 8.2 +   │   │  DB GUI        │  │
    │  :30081 (pma)     │  │  Apache      │   │  :30081        │  │
    └──────────────────►│  └──────┬───────┘   └──────┬─────────┘  │
                        │         │  ClusterIP         │           │
                        │         └──────────┬─────────┘           │
                        │                    ▼                      │
                        │           ┌────────────────┐             │
                        │           │  mysql (x1)    │             │
                        │           │  MySQL 8.0     │             │
                        │           │  + PVC (1Gi)   │             │
                        │           └────────────────┘             │
                        └─────────────────────────────────────────┘
```

**Services exposed:**
| Service       | Type     | Port  | URL                             |
|---------------|----------|-------|---------------------------------|
| carrental-web | NodePort | 30080 | `http://<node-ip>:30080`        |
| phpmyadmin    | NodePort | 30081 | `http://<node-ip>:30081`        |
| mysql         | ClusterIP| 3306  | Internal only (no external access)|

---

## Project Structure

```
car-rental-k8s/
├── .github/
│   └── workflows/
│       └── ci-cd.yml           # GitHub Actions pipeline (lint → build/push → deploy)
├── .gitignore                  # Excludes .claude/ and OS/editor junk
├── app/                        # PHP web application
│   ├── Dockerfile              # Builds the PHP + Apache image
│   ├── index.php               # Homepage
│   ├── car-listing.php         # Vehicle catalog
│   ├── vehical-details.php     # Vehicle detail page
│   ├── my-booking.php          # User bookings
│   ├── admin/                  # Admin panel
│   ├── includes/               # Shared PHP includes (config, header, footer)
│   └── assets/                 # CSS, JS, images
│
├── mysql/
│   ├── Dockerfile              # MySQL image with database pre-loaded
│   └── carrental.sql           # Full database dump (schema + sample data)
│
├── k8s/                        # Kubernetes manifests (apply in order)
│   ├── 00-namespace.yaml       # Namespace: carrental
│   ├── 01-secret.yaml          # DB credentials (base64)
│   ├── 02-configmap.yaml       # DB host and name
│   ├── 03-mysql-pvc.yaml       # Persistent storage for MySQL (1Gi)
│   ├── 04-mysql-deployment.yaml# MySQL pod
│   ├── 05-mysql-service.yaml   # MySQL ClusterIP service
│   ├── 06-web-deployment.yaml  # PHP/Apache pods (2 replicas)
│   ├── 07-web-service.yaml     # Web NodePort service (:30080)
│   ├── 08-phpmyadmin-deployment.yaml
│   ├── 09-phpmyadmin-service.yaml  # phpMyAdmin NodePort (:30081)
│   └── 10-ingress.yaml         # Optional: domain-based routing
│
└── README.md
```

---

## Push This Repo to GitHub

This repo has a remote already configured (`origin` → `https://github.com/chetan080808/Carrental_Ci-CD.git`) but **no commits yet** — none of this has been pushed. A `.gitignore` excludes the local `.claude/` folder (Claude Code's machine-specific settings) so it never reaches GitHub.

```bash
git add .
git commit -m "Initial commit: Docker + K8s + CI/CD setup"
git push -u origin main
```

If `.claude/` was ever committed in an earlier attempt, untrack it first (it's already excluded going forward, this just removes it from what's already staged/tracked):
```bash
git rm -r --cached .claude
```

Once pushed, add the `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` repo secrets and set up the self-hosted runner (see [CI/CD with GitHub Actions](#cicd-with-github-actions) below) so the pipeline can build, push, and deploy on every merge to `main`.

---

## Prerequisites

Install these tools on your machine before starting:

| Tool       | Purpose                              | Download                                     |
|------------|--------------------------------------|----------------------------------------------|
| Docker     | Build and push container images      | https://docs.docker.com/get-docker/          |
| kubectl    | Kubernetes command-line tool         | https://kubernetes.io/docs/tasks/tools/      |
| Minikube   | Run Kubernetes locally (recommended) | https://minikube.sigs.k8s.io/docs/start/     |

> **Alternatives to Minikube:** kind, Docker Desktop Kubernetes, or any cloud cluster (GKE, EKS, AKS).

### Verify installations

```bash
docker --version
kubectl version --client
minikube version
```

---

## Step 1 — Start Minikube

```bash
minikube start --driver=docker --memory=2048 --cpus=2
```

Check it is running:

```bash
minikube status
kubectl get nodes
```

You should see one node with status `Ready`.

---

## Step 2 — Build Docker Images

You need two custom images:
1. **Web image** — PHP + Apache with the application code
2. **MySQL image** — MySQL pre-loaded with the `carrental` database

### 2a. Build the web image

```bash
docker build -t chetan0808/carrental-web:latest ./app
```

### 2b. Build the MySQL image

```bash
docker build -t chetan0808/carrental-mysql:latest ./mysql
```

Replace `chetan0808` with your actual Docker Hub username (e.g. `john123`).

---

## Step 3 — Push Images to Docker Hub

Docker Hub is a free public registry. Images pushed here can be pulled by Kubernetes.

### Login

```bash
docker login
```

### Push both images

```bash
docker push chetan0808/carrental-web:latest
docker push chetan0808/carrental-mysql:latest
```

---

## Step 4 — Update Image Names in Manifests

Open these two files and replace `chetan0808` with your actual username:

**`k8s/04-mysql-deployment.yaml`** — line with `image:`:
```yaml
image: chetan0808/carrental-mysql:latest
```

**`k8s/06-web-deployment.yaml`** — line with `image:`:
```yaml
image: chetan0808/carrental-web:latest
```

---

## Step 5 — Deploy to Kubernetes

Apply all manifests in order using a single command:

```bash
kubectl apply -f k8s/
```

This applies all YAML files in the `k8s/` folder in alphabetical order (00 → 10).

### Verify everything is running

```bash
# Watch pods come up (Ctrl+C to stop watching)
kubectl get pods -n carrental -w

# Check all resources
kubectl get all -n carrental
```

Expected output (all pods should show `Running`):
```
NAME                                READY   STATUS    RESTARTS   AGE
pod/mysql-xxxxxxxxxx-xxxxx          1/1     Running   0          2m
pod/carrental-web-xxxxxxxxxx-xxxxx  1/1     Running   0          90s
pod/carrental-web-xxxxxxxxxx-yyyyy  1/1     Running   0          90s
pod/phpmyadmin-xxxxxxxxxx-xxxxx     1/1     Running   0          90s

NAME                    TYPE        CLUSTER-IP       PORT(S)
service/mysql           ClusterIP   10.96.xxx.xxx    3306/TCP
service/carrental-web   NodePort    10.96.xxx.xxx    80:30080/TCP
service/phpmyadmin      NodePort    10.96.xxx.xxx    80:30081/TCP
```

---

## Step 6 — Access the Application

### Get the Minikube node IP

```bash
minikube ip
```

Example output: `192.168.49.2`

### Open in your browser

| Page              | URL                               |
|-------------------|-----------------------------------|
| Car Rental Portal | `http://192.168.49.2:30080`       |
| Admin Panel       | `http://192.168.49.2:30080/admin` |
| phpMyAdmin        | `http://192.168.49.2:30081`       |

> **Tip:** Minikube also has a shortcut:
> ```bash
> minikube service carrental-web -n carrental
> ```
> This opens the browser automatically.

### Default Credentials

**Admin Panel** (`/admin`):
- Username: `admin`
- Password: `Test@123`

**phpMyAdmin** (database GUI):
- Server: `mysql`
- Username: `root`
- Password: `rootpassword`

---

## Step 7 — Optional: Use Ingress (Domain-Based Access)

Instead of using IP:port, you can access the app via a custom domain.

### Enable the Ingress addon

```bash
minikube addons enable ingress
```

### Apply the Ingress manifest

```bash
kubectl apply -f k8s/10-ingress.yaml
```

### Add hosts entries

Get the Minikube IP:
```bash
minikube ip
```

**Windows** — edit `C:\Windows\System32\drivers\etc\hosts` as Administrator:
```
192.168.49.2  carrental.local
192.168.49.2  pma.carrental.local
```

**Linux / Mac** — edit `/etc/hosts`:
```bash
sudo nano /etc/hosts
# Add:
192.168.49.2  carrental.local
192.168.49.2  pma.carrental.local
```

### Access via domain

| Page              | URL                               |
|-------------------|-----------------------------------|
| Car Rental Portal | `http://carrental.local`          |
| phpMyAdmin        | `http://pma.carrental.local`      |

---

## Kubernetes Concepts Explained

### What is a Namespace?
A namespace is like a folder that groups related resources together.  
All resources in this project live in the `carrental` namespace.

### What is a Deployment?
A Deployment ensures your pods are always running. If a pod crashes,
Kubernetes automatically restarts it. It also manages rolling updates.

### What is a Service?
A Service gives a stable network address to a pod (or group of pods).
Pods are temporary; services are permanent.

- **ClusterIP** — internal only (used for MySQL — never exposed externally)
- **NodePort** — accessible from outside on a fixed port (30000–32767)
- **LoadBalancer** — cloud provider gives a public IP (used in GKE, EKS, AKS)

### What is a PersistentVolumeClaim (PVC)?
A PVC reserves disk space for a pod. Without it, MySQL data is lost
when the pod restarts. The PVC keeps data safe across pod restarts.

### What is a Secret?
A Secret stores sensitive values like passwords.  
Values are base64-encoded (NOT encrypted by default — use Vault or Sealed Secrets in production).

### What is a ConfigMap?
A ConfigMap stores non-sensitive configuration like hostnames and database names.

### What is an InitContainer?
An init container runs and completes before the main container starts.  
In this project, it waits for MySQL to be ready before the PHP app starts.

---

## Useful kubectl Commands

```bash
# See all resources in the namespace
kubectl get all -n carrental

# Watch pods in real-time
kubectl get pods -n carrental -w

# View logs from the web pod
kubectl logs -n carrental deployment/carrental-web

# View logs from MySQL
kubectl logs -n carrental deployment/mysql

# Describe a pod (useful for debugging)
kubectl describe pod -n carrental <pod-name>

# Open a shell inside the web container
kubectl exec -it -n carrental deployment/carrental-web -- bash

# Open a MySQL shell
kubectl exec -it -n carrental deployment/mysql -- mysql -u root -prootpassword carrental

# Scale web pods up or down
kubectl scale deployment carrental-web -n carrental --replicas=3

# Delete and re-create all resources
kubectl delete -f k8s/
kubectl apply -f k8s/
```

---

## Changing the Database Password

1. Edit `k8s/01-secret.yaml`. Encode your new password:

```bash
echo -n 'MyNewPassword' | base64
```

2. Replace the values in the file.

3. Re-apply:
```bash
kubectl apply -f k8s/01-secret.yaml
kubectl rollout restart deployment/mysql -n carrental
kubectl rollout restart deployment/carrental-web -n carrental
```

---

## Troubleshooting

### Pod is in `Pending` state
```bash
kubectl describe pod <pod-name> -n carrental
```
Usually caused by:
- Not enough CPU/memory — increase Minikube resources: `minikube start --memory=4096`
- PVC not bound — check `kubectl get pvc -n carrental`

### Pod is in `ImagePullBackOff` or `ErrImagePull`
The image cannot be pulled from Docker Hub.  
- Check the image name in the deployment YAML matches exactly what you pushed
- Make sure you ran `docker push` successfully
- Check Docker Hub: `https://hub.docker.com/u/chetan0808`

### Web app shows database connection error
MySQL might not be ready yet. Wait 30–60 seconds and refresh.  
Check MySQL logs:
```bash
kubectl logs -n carrental deployment/mysql
```

### Cannot access `http://<ip>:30080`
- Confirm the pod is `Running`: `kubectl get pods -n carrental`
- On Minikube, use: `minikube service carrental-web -n carrental`

### Ingress not working
```bash
kubectl get ingress -n carrental
kubectl describe ingress carrental-ingress -n carrental
```
Make sure the Ingress addon is enabled: `minikube addons enable ingress`

---

## Tear Down

To stop and delete everything:

```bash
# Delete all K8s resources
kubectl delete -f k8s/

# Stop Minikube
minikube stop

# Delete Minikube cluster (removes all data)
minikube delete
```

---

## CI/CD with GitHub Actions

The pipeline lives at `.github/workflows/ci-cd.yml` and has three jobs:

| Job                 | Runs on           | Trigger                          | What it does |
|---------------------|-------------------|-----------------------------------|--------------|
| `lint-and-validate`  | GitHub-hosted     | Every push & PR to `main`        | `php -l` syntax-checks every file in `app/`, and `kubeconform` validates every manifest in `k8s/` against the Kubernetes schema. |
| `build-and-push`     | GitHub-hosted     | Push to `main` only (not PRs)    | Builds `app/Dockerfile` and `mysql/Dockerfile`, tags each image `latest` + `sha-<commit>`, pushes both to Docker Hub. |
| `deploy`             | **Self-hosted** (your EC2 box) | After `build-and-push` succeeds | `kubectl apply -f k8s/`, then pins `carrental-web` to the freshly-built `sha-<commit>` image and waits for the rollout to finish. |

A PR only runs lint/validate. Merging to `main` builds, pushes, and deploys.

### One-time setup

**1. Repo secrets** (Settings → Secrets and variables → Actions):
| Secret | Value |
|---|---|
| `DOCKERHUB_USERNAME` | Your Docker Hub username |
| `DOCKERHUB_TOKEN` | A Docker Hub [access token](https://hub.docker.com/settings/security) (not your password) |

**2. Self-hosted runner on your EC2 instance**, where the Kubernetes cluster (Minikube/k3s/kind) also runs:
```bash
# On the EC2 box
mkdir actions-runner && cd actions-runner
curl -o actions-runner.tar.gz -L https://github.com/actions/runner/releases/latest/download/actions-runner-linux-x64.tar.gz
tar xzf actions-runner.tar.gz
./config.sh --url https://github.com/<you>/<repo> --token <token-from-repo-settings>
sudo ./svc.sh install
sudo ./svc.sh start
```
Get the registration token from **Settings → Actions → Runners → New self-hosted runner**. Leave the default label (`self-hosted`) — the workflow targets it as-is.

The runner's user needs a working `kubectl` pointed at your cluster (`kubectl config current-context` should succeed) — same shell/user the runner service runs as.

**3. (Optional) Protect the deploy step** — create a GitHub **Environment** named `production` (Settings → Environments) and add required reviewers. The `deploy` job already targets this environment, so once configured, every deploy waits for manual approval before touching the cluster.

> Note: `k8s/04-mysql-deployment.yaml` currently runs the stock `mysql:8.0` image and loads data via an init container that curls `carrental.sql` from GitHub — it doesn't consume the `mysql/` custom image. The pipeline still builds/pushes it for parity with local `docker-compose`, but deploy only rolls out `carrental-web`. Say the word if you'd rather have the mysql Deployment use the built image instead.

---

## Next Steps

Once you are comfortable with this setup, explore:

- **Helm Charts** — package your K8s manifests as a reusable chart
- **Horizontal Pod Autoscaler (HPA)** — auto-scale based on CPU/memory
- **Liveness & Readiness Probes** — already configured in this project, read more in the K8s docs
- **Resource Limits** — add `resources.requests` and `resources.limits` to each container
- **Sealed Secrets / Vault** — proper secrets management for production
- **Cloud Deployment (AWS)** — migrate this same setup to EKS, ECR, and RDS
