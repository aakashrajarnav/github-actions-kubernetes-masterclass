# SkillPulse — GitHub Actions & Kubernetes Masterclass

A small, real application with a real CI/CD pipeline. The app — SkillPulse — lets you track skills you're learning and the hours you put in. The point isn't the app. The point is everything around it: how a single `git push` becomes a running update on a server in under two minutes, with no human pressing any button.

## CI/CD pipeline

### CI — `.github/workflows/ci.yml`

Triggered on every push to `main`. Steps:

1. **Checkout** the code in a clean Ubuntu runner.
2. **Build two Docker images** — a Go backend and an Nginx-served frontend. Both are multi-stage so final images are small.
3. **Tag each image twice** — with the commit SHA (`:abc1234…`) for rollback precision, and `:latest` for production pulls.
4. **Push both to Docker Hub** — authenticated with secrets, never plaintext credentials.

From the latest run: `frontend` built in **6s** (0% cached), `backend` built in **5s** (44% cached). Both completed as part of a `build-and-push` job that finished in **46s** total.

### CD — GitOps with ArgoCD

Rather than SSH-deploying directly, the CD pipeline follows the **GitOps** pattern:

1. CI builds and pushes the new images.
2. The CD workflow bumps the image tag in the Kubernetes manifests and commits the change back to `main`.
3. **ArgoCD** running in the cluster detects the manifest drift and automatically syncs the cluster state — no manual `kubectl apply` needed.

**Current ArgoCD state:**
- App health: ✅ **Healthy**
- Sync status: ⚡ **OutOfSync** from `main (fc2cab3)` — auto-sync is enabled and will reconcile
- Last sync: ✅ **Sync OK** to `fc2cab3`, succeeded at 01:53:52 (commit: `ci: update k8s images to 95bb4d6 [skip ci]`)

The ArgoCD tree shows `frontend` and `backend` Deployments with 6 ReplicaSets each (rolling history), a `mysql` StatefulSet Service, and a running frontend pod (`frontend-b96b657b9-fchqh`).

### Secrets used

| Secret | What it is |
|---|---|
| `DOCKERHUB_USERNAME` | Docker Hub account name |
| `DOCKERHUB_TOKEN` | Docker Hub Personal Access Token with read+write scope |
| `EC2_HOST` | Public IP or DNS of the deploy target |
| `EC2_USER` | Linux user on the EC2 (typically `ubuntu`) |
| `EC2_SSH_KEY` | Private key contents — paste the entire `.pem` file as the secret value |

Set them at `Settings → Secrets and variables → Actions` on your fork.

---

## Infrastructure

### AWS EKS cluster

```
Cluster:   skillpulse-eks
Region:    ap-south-1 (Mumbai)
Version:   Kubernetes 1.35
Status:    Active
Support:   Standard support until March 27, 2027
Provider:  EKS
Created:   ~3 hours before screenshots (May 20 2026)
```

### EC2 worker nodes

| Name | Instance ID | Type | Availability Zone |
|---|---|---|---|
| skillpulse-ng | i-039eff50ccdafd5a4 | m7i-flex.large | ap-south-1b |
| skillpulse-ng | i-0b7187fd052144c31 | m7i-flex.large | ap-south-1c |
| skillpulse-ng | i-0f84f6bab07573c9f | m7i-flex.large | ap-south-1a |

All instances are **Running** with 3/3 status checks passed.

### Docker Hub

Two public repositories, both pushed **13 minutes before screenshots**:

- [`aakashrajarnav/skillpulse-frontend`](https://hub.docker.com/r/aakashrajarnav/skillpulse-frontend)
- [`aakashrajarnav/skillpulse-backend`](https://hub.docker.com/r/aakashrajarnav/skillpulse-backend)

---

## Observability

### Node metrics (Grafana + Node Exporter)

Prometheus and Grafana are deployed in the cluster. The Node Exporter Full dashboard shows the worker node (`ip-10-0-4-82.ap-south-1.compute.internal`) at very low utilisation — healthy headroom:

| Metric | Value |
|---|---|
| CPU (system) | 0.7% |
| CPU (overall) | 3.2% |
| Memory used | 17.6% |
| Root fs used | 33.2% |
| Uptime | 2.4 hours |
| RAM | 8 GiB |
| Storage | 20 GiB |

Network traffic and disk space charts show clean baselines with no anomalies.

### Kubernetes API Server (Grafana SLOs)

The Kubernetes / API Server dashboard reports against 30-day SLO windows:

| SLO | Value |
|---|---|
| Availability (30d) | **100.000%** |
| Read Availability (30d) | **100.000%** |
| Write Availability (30d) | **100.000%** |
| Error Budget (30d) | > 99.000% — fully intact |

Read and Write SLI request rates, error rates, and durations are all within bounds. No data anomalies in the error panels.

---

## The application itself

A three-tier app — kept small on purpose so the pipeline is the star.

| Tier | Tech | What it does |
|---|---|---|
| Frontend | HTML + CSS + vanilla JS, served by Nginx | UI for adding skills and logging hours |
| Backend | Go 1.26 + Gin | REST API at `/api/...` |
| Database | MySQL 8.4 | Stores skills and learning logs |

Nginx in the frontend image also reverse-proxies `/api/` and `/health` to the backend, so the public surface is a single port (`80`).

**Live stats (from the app on May 20 2026):**

| Stat | Value |
|---|---|
| Total Skills | 16 |
| Hours Logged | 48.0 |
| Sessions | 27 |
| Top Skill | Docker |

API surface:

```
GET    /api/skills              list skills + total hours
POST   /api/skills              create skill
GET    /api/skills/:id          one skill + its logs
DELETE /api/skills/:id          delete skill (cascades logs)
POST   /api/skills/:id/log      log a study session
GET    /api/dashboard           summary counters
GET    /health                  DB ping for healthchecks
```

---

## Why GitHub Actions

A pipeline needs a runner — something that watches your repo, executes your build/test/deploy steps, and reports back. Historically that meant standing up a Jenkins server, paying for CircleCI, or wiring something custom. All of those still work; none of them are the lowest-friction option in 2026.

GitHub Actions wins on three things:

1. **It lives where the code lives.** No separate server, no separate auth, no separate UI. Your `.github/workflows/*.yml` files are part of the repo — they evolve with the code, get reviewed in the same PRs, and survive every clone.
2. **It's free for public repos and generous for private ones.** A complete CI/CD pipeline costs zero rupees to start.
3. **The Marketplace is enormous.** Need to SSH into a server? `appleboy/ssh-action`. Need to log in to Docker Hub? `docker/login-action`. You compose pre-built blocks instead of writing bash from scratch.

The trade-off is GitHub lock-in. For most teams, that's a fair price for the integration.

---

## What this project demonstrates

A real pipeline, end to end, in roughly 50 lines of YAML.

```
┌─────────────┐     git push        ┌──────────────────┐
│  Developer  ├────────────────────▶│  GitHub Repo     │
└─────────────┘                     └────────┬─────────┘
                                             │ on: push (main)
                                             ▼
                                    ┌──────────────────┐
                                    │  CI Workflow     │
                                    │  - build images  │
                                    │  - tag :sha      │
                                    │  - tag :latest   │
                                    │  - push to Hub   │
                                    └────────┬─────────┘
                                             │ workflow_run: success
                                             ▼
                                    ┌──────────────────┐
                                    │  CD Workflow     │
                                    │  - SSH to EC2    │
                                    │  - git pull      │
                                    │  - compose pull  │
                                    │  - compose up -d │
                                    └────────┬─────────┘
                                             │
                                             ▼
                                    ┌──────────────────┐
                                    │  EC2: live app   │
                                    │  http://<host>   │
                                    └──────────────────┘
```

### CI — `.github/workflows/ci.yml`

Triggered on every push to `main`. It does four things:

1. **Checks out the code.** A fresh clone in a clean Ubuntu runner — no laptop state to leak.
2. **Builds two Docker images.** A Go backend and an Nginx-served frontend. Both are multi-stage so the final images are small.
3. **Tags each image twice.** With the commit SHA (`:abc1234…`) and with `:latest`. The SHA tag is your rollback handle — you can always pin a deploy to an exact commit. The `:latest` tag is what production pulls.
4. **Pushes both to Docker Hub.** Authenticated with secrets (`DOCKERHUB_USERNAME`, `DOCKERHUB_TOKEN`) — never plaintext credentials in the repo.

The non-obvious lesson: **CI doesn't just test your code. It produces an artifact.** That artifact — the image — is what production runs. If the artifact is built consistently in CI, it's the same in dev, staging, and prod. "Works on my machine" stops being a possibility.

---

## Run it locally

```bash
cp .env.example .env             # fill in DOCKERHUB_USERNAME (anything works for local)
docker compose up -d --build
```

Open http://localhost. Backend port 8080 is intentionally not exposed — all traffic goes through Nginx, exactly like production.

To tear down:

```bash
docker compose down -v           # -v also drops the MySQL volume
```

---

## Run on Kubernetes (kind)

Same app, same images, same external port — but now every primitive a student would see in production: namespace, deployment, service, statefulset, configmap, secret, pvc.

**Prerequisites:** Docker Desktop running, plus `brew install kind kubectl`.

```bash
make up                          # creates the kind cluster + applies manifests
# visit http://localhost:8888
make down                        # deletes the cluster (and the MySQL data with it)
```

What `make up` actually runs, in order:

```bash
docker build -t trainwithshubham/skillpulse-backend:latest  ./backend
docker build -t trainwithshubham/skillpulse-frontend:latest ./frontend
kind create cluster --config k8s/kind-config.yaml --name skillpulse
kind load docker-image trainwithshubham/skillpulse-backend:latest  --name skillpulse
kind load docker-image trainwithshubham/skillpulse-frontend:latest --name skillpulse
kubectl apply -f k8s/00-namespace.yaml \
              -f k8s/10-mysql.yaml \
              -f k8s/20-backend.yaml \
              -f k8s/30-frontend.yaml
kubectl rollout status statefulset/mysql   -n skillpulse --timeout=180s
kubectl rollout status deployment/backend  -n skillpulse --timeout=120s
kubectl rollout status deployment/frontend -n skillpulse --timeout=60s
```

Notes on this flow:

- **`docker build` runs on your laptop**, producing images for your host's architecture (Apple Silicon → arm64; Intel/Linux → amd64). The cluster never has to deal with multi-arch.
- **`kind load docker-image`** copies each image into the kind node's containerd. `imagePullPolicy: IfNotPresent` on the Deployments means k8s reuses the loaded image and never tries to pull from Docker Hub.
- **`kind-config.yaml`** lives alongside the manifests for proximity, but it's a `kind` config — not a Kubernetes resource — so it's fed to `kind create cluster`, not `kubectl apply`.

Inner-loop after editing code: `make restart` rebuilds the images, reloads them into the cluster, and rolls the Deployments.

### How traffic flows

The cluster has **three nodes**: one control-plane and two workers (`skillpulse-worker`, `skillpulse-worker2`). Workloads schedule onto the workers — the control-plane is tainted `NoSchedule` by default, so it stays focused on the API server, scheduler, and controller-manager.

```
host browser            kind cluster (1 control-plane + 2 workers)
http://localhost:8888
        │
        ▼ (kind extraPortMappings on control-plane: hostPort 8888 → nodePort 30080)
   Service frontend (NodePort 30080)  — reachable on every node, kube-proxy routes
        │
        ▼
   Deployment frontend (nginx + static)  — runs on whichever worker the scheduler picks
        │ proxy_pass http://backend:8080  (same hostname as docker-compose)
        ▼
   Service backend (ClusterIP 8080)
        │
        ▼
   Deployment backend (Go + Gin)
        │ DB_HOST=mysql
        ▼
   Service mysql (Headless 3306)
        │
        ▼
   StatefulSet mysql + 1Gi PVC + ConfigMap-mounted init.sql
```

### Manifest layout

```
k8s/
  kind-config.yaml      cluster shape: 1 control-plane + 2 workers, host 8888 → node 30080
  00-namespace.yaml     namespace: skillpulse
  10-mysql.yaml         Secret + ConfigMap (init.sql) + headless Service + StatefulSet + 1Gi PVC
  20-backend.yaml       Deployment + ClusterIP Service, env from Secret, /health probes
  30-frontend.yaml      Deployment + NodePort Service (30080), / probes
```

### Useful commands

| Command | What it does |
|---|---|
| `make status` | One-screen view of pods, services, endpoints |
| `make logs` | Tail all three workloads at once |
| `make mysql` | Open a `mysql` shell in the StatefulSet pod |
| `make restart` | Roll backend + frontend (e.g. after pushing a new image) |

### Smoke test

```bash
curl http://localhost:8888/health                 # → {"status":"healthy"}
curl http://localhost:8888/api/dashboard          # → seed-data counters
curl -s http://localhost:8888/ | grep '<title>'   # → HTML title containing "SkillPulse"
```

### Gotchas worth knowing

- **Docker Desktop must be running.** `docker build`, `kind`, and `kubectl` all talk to the Docker daemon on your machine.
- **First boot is slow.** The local-path provisioner has to materialise the PVC before MySQL starts. Expect 10–30s of `Pending` on `make up`'s first run.
- **Host port collision.** If something else owns 8888 on the host, the cluster comes up but `curl localhost:8888` fails. Free the port — or change `hostPort` in `k8s/kind-config.yaml` and re-run `make down && make up`.
- **No Docker Hub round-trip in this chapter.** Images are built locally and pushed into the kind node via `kind load`. Useful when you're iterating on code: `make restart` rebuilds + reloads + rolls without ever touching Docker Hub. (Production EKS/GKE clusters do pull from a registry — that's the next chapter.)

### What's next

This is the **kind chapter** — same app, real Kubernetes primitives, but limited to one local node and `NodePort` access. The next chapter graduates the same workload to:

- An **Ingress** controller (nginx-ingress) so traffic enters via `Ingress` rules instead of NodePort.
- **Helm or Kustomize** so the manifests stop being copy-pasted between environments.
- A real **cloud cluster** (EKS / GKE / AKS) and CD that runs `kubectl apply` from the pipeline instead of `appleboy/ssh-action`.

---

## Continuous deployment to the kind cluster

The new CD path doesn't `kubectl apply` from GitHub Actions — your kind cluster lives on your laptop, GitHub can't reach it. Instead, the pipeline takes the GitOps shape: **the repo is the source of truth, your cluster is one `git pull && make apply` away**.

```
git push to main
    ↓
CI: build images, push trainwithshubham/skillpulse-{backend,frontend}:{latest,<sha>}
    ↓
cd-k8s.yml: sed image: lines in k8s/20-backend.yaml + k8s/30-frontend.yaml
            commit "deploy: pin backend+frontend to <short-sha>" to main as github-actions[bot]
    ↓
(you, locally):
    git pull && make apply
    ↓
kind nodes pull the new :<sha> from Docker Hub → rolling update
```

### How to wire it up on your fork

1. **Fork this repo + clone locally.** `make up` should work after that (see the [Run on Kubernetes (kind)](#run-on-kubernetes-kind) section).
2. **Add two secrets** to your fork (`Settings → Secrets and variables → Actions`):

   | Secret | Value |
   |---|---|
   | `DOCKERHUB_USERNAME` | your Docker Hub account name |
   | `DOCKERHUB_TOKEN` | a Docker Hub Personal Access Token with Read & Write scope |

3. **Set the repo variable** `DEPLOY_ENABLED = "true"` (`Settings → Variables → Actions`). Until this is `true`, CI builds without pushing and both CD workflows skip cleanly — the "dry run" state.
4. **Push any code change** (not a `.md`, not under `k8s/` or `docs/` — those are deliberately ignored by CI). Watch the Actions tab:
   - **CI** builds + pushes both images to Docker Hub.
   - **CD (kind cluster — manifest bump)** commits a `deploy: pin backend+frontend to <sha>` change to main.
5. **Pull and deploy**, on the laptop with the kind cluster:
   ```bash
   git pull
   make apply
   kubectl get pods -n skillpulse -o wide
   ```
   You'll see new pods with the bumped image rolling out. mysql untouched.

### What about the EC2 path?

The previous chapter's `cd.yml` is still in the repo — it SSHes into an EC2 and runs `docker compose up`. It's gated on the same `DEPLOY_ENABLED` variable plus three EC2 secrets (`EC2_HOST`, `EC2_USER`, `EC2_SSH_KEY`). Skip those secrets and `cd.yml` will fail loudly when `DEPLOY_ENABLED=true`; that's expected — it's the previous chapter's deploy target, kept around as the masterclass artifact.

### Break it on purpose to learn

- **Push a commit that fails to build** → both CD workflows are *skipped*, not failed (the `if: success()` gate).
- **Rotate the Docker Hub token** → next CI fails at the login step. You'll learn what an expired credential looks like in logs.
- **Edit `k8s/20-backend.yaml`'s image tag by hand and push** → CI is *skipped* (paths-ignore), `cd-k8s.yml` does fire but the manifest is already pinned, so it no-ops and exits 0. That's the loop-protection working.

---

## Project layout

```
backend/                Go service
  Dockerfile            multi-stage: golang:1.26-alpine → alpine:3.23
  main.go               wires routes, reads PORT env
  database/db.go        connects to MySQL with retry-loop
  handlers/             skills, logs, dashboard endpoints
  models/               request/response structs

frontend/               static UI + Nginx config
  Dockerfile            FROM nginx:alpine, copies html/css/js + nginx.conf
  index.html, css/, js/ vanilla — no build step
  nginx.conf            serves the site, proxies /api/ to backend:8080

mysql/init.sql          schema + seed data, mounted into the MySQL container

terraform/
  .terraform.lock.hcl
  argocd.tf
  eks.tf
  outputs.tf
  provider.tf
  terraform.tfvars
  variables.tf
  vpc.tf

k8s/
  00-namespace.yaml
  10-mysql.yaml
  20-backend.yaml
  30-frontend.yaml
  cert-manager.yml
  gateway.yml
  secrets.yml

docker-compose.yml      three services: db, backend, frontend
.env.example            copy to .env

.github/workflows/
  ci.yml                build + push images on every main push
```

## Credits

Built for the [TrainWithShubham](https://www.youtube.com/@TrainWithShubham) community. If this repo helped you understand a real CI/CD pipeline end to end, share it forward — that's how the community grows.
