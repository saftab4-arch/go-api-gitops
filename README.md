# Two-Repo GitOps CI/CD — Go API with GitHub Actions + Argo CD

A complete GitOps pipeline where **a single `git push` of code ends with a new version running in Kubernetes** — no `docker build`, no `docker push`, no `kubectl`, no manual deploy. GitHub Actions handles **CI** (build the image, push it, update the config repo). Argo CD handles **CD** (watch the config repo, deploy to the cluster). CI never touches the cluster and holds zero cluster credentials.

This README is written so anyone can replicate the project from scratch. Every command, file, and design decision is explained.

---

## The core idea: CI and CD are separate, and so are the repos

Two clean separations drive the whole design:

**CI vs CD.** GitHub Actions builds and publishes the artifact and updates the desired state in Git. It never deploys. Argo CD, running inside the cluster, watches Git and does all deploying. CI produces; CD reconciles.

**Two repos.** The app code lives in one repo; the Kubernetes manifests live in another. This isn't tidiness — it's what prevents an infinite loop (explained below) and cleanly separates *what the app is* from *what runs*.

```
Repo 1 — APP CODE (go-api-gitops)      Repo 2 — CONFIG (go-api-config)
  ├── main.go                            ├── application.yaml   (Argo CD Application)
  ├── Dockerfile                         └── manifests/
  └── .github/workflows/ci.yaml                ├── deployment.yaml  (image: ...:TAG)
                                               └── service.yaml
```

---

## The full automated flow

```
1. Developer edits main.go, git push          -> Repo 1 (app)
        |  (push triggers the workflow: "on: push")
        v
2. GitHub Actions (CI) runs on a cloud VM:
     - checkout code
     - docker build   (using the existing Dockerfile)
     - docker push    -> Docker Hub  (syeddocker04/go-api:<commit-sha>)
     - sed the image tag in Repo 2's deployment.yaml -> commit & push
        |
        v  (Argo CD watches Repo 2, NOT Docker Hub)
3. Argo CD (CD) sees the config commit
     - syncs the new image into the cluster (rolling update)
        |
        v
4. New version live — zero manual steps
```

**Why the tag-update step exists:** Argo CD only watches the **config repo (Git)**. Pushing an image to Docker Hub is invisible to it. The *only* thing that makes Argo CD act is a change to the manifest it watches — so CI must edit the `image:` tag in the config repo. That commit is the bridge between "image built" and "image deployed."

**Why two repos (the loop):** if CI committed the tag change back into the *same* repo that triggered it, the push would re-trigger the pipeline forever. Writing the tag to a *separate* repo breaks the loop. This is the concrete reason the repos are split.

---

## The port chain (browser -> pod)

```
browser -> localhost:80      (kind hostPort mapping in kind-config.yaml)
        -> node:30950        (NodePort — must match service.yaml nodePort)
        -> service:8080      (Service port)
        -> go-api pod:8080   (targetPort / containerPort / Go ListenAndServe)
```

The Go app listens on **8080**, so the container port, service `targetPort`, and service `port` are all 8080. The `nodePort` stays **30950** to match the kind host mapping.

---

## Prerequisites

| Tool | Check |
|------|-------|
| Docker | `docker ps` |
| kind | `kind version` |
| kubectl | `kubectl version --client` |
| git | `git --version` |
| Go (optional, for local test) | `go version` |
| Docker Hub account | — |
| GitHub account | — |

---

## Phase 0 — Cluster + Argo CD

### Create the kind cluster

**`kind-config.yaml`:**

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    image: kindest/node:v1.31.2
    extraPortMappings:
      - containerPort: 30950
        hostPort: 80
        protocol: TCP
  - role: worker
    image: kindest/node:v1.31.2
```

```bash
kind create cluster --name gitops-demo --config kind-config.yaml
kubectl get nodes          # wait for both Ready
```

### Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

> **Why `--server-side`:** a plain `kubectl apply` fails on the `applicationsets` CRD with `Too long: must have at most 262144 bytes`. Client-side apply stores a full copy of each resource in a `last-applied-configuration` annotation, and Kubernetes caps annotations at 256KB — that CRD's schema is too big. Server-side apply tracks field ownership on the API server instead of writing that annotation, so the limit never applies. It's the recommended install method.

Verify and log in:

```bash
kubectl get pods -n argocd                 # all 7 Running 1/1
kubectl get crd | grep argoproj            # 3 CRDs

# admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# UI (use 8081 locally to avoid clashing with the app's 8080 later)
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

Open **https://localhost:8081**, log in as `admin` + the decoded password.

---

## Phase 1 — The Go app + image

Create two **public**, **empty** GitHub repos: `go-api-gitops` (app) and `go-api-config` (manifests).

### The app

**`main.go`:**

```go
package main

import (
	"fmt"
	"net/http"
)

func handler(w http.ResponseWriter, r *http.Request) {
	fmt.Fprintln(w, "Hello from Syed's Go API! Version 1")
}

func main() {
	http.HandleFunc("/", handler)
	fmt.Println("Server starting on port 8080...")
	http.ListenAndServe(":8080", nil)
}
```

The app is a minimal HTTP server: `handler` writes a line into the response, `main` wires `/` to it and listens on 8080. The `"Version 1"` string is the proof-of-deploy marker — changing it later demonstrates the whole pipeline.

### The multi-stage Dockerfile

**`Dockerfile`:**

```dockerfile
# ---- Stage 1: build ----
FROM golang:1.23 AS builder
WORKDIR /app
COPY main.go .
RUN CGO_ENABLED=0 GOOS=linux go build -o server main.go

# ---- Stage 2: run ----
FROM alpine:3.20
WORKDIR /app
COPY --from=builder /app/server .
EXPOSE 8080
CMD ["./server"]
```

**Why two stages:** compiling Go needs the full ~800MB toolchain (`golang:1.23`); *running* the compiled binary needs almost nothing. Stage 1 builds a fully static binary (`CGO_ENABLED=0` = no C dependencies, `GOOS=linux` = target Linux). Stage 2 starts fresh from tiny Alpine (~5MB) and copies **only** the binary across (`COPY --from=builder`). Result: a **~8MB** final image instead of ~800MB.

> Alpine is a minimal Linux distribution, not a web server. The Go binary *is* the web server (`ListenAndServe`). Alpine is just a thin base to run it on.

### Build and push (by hand, once)

```bash
docker build -t syeddocker04/go-api:v1 .
docker images | grep go-api          # note the tiny CONTENT SIZE (~8MB)

docker login                          # username + Docker Hub access token
docker push syeddocker04/go-api:v1
```

### Push the app repo

```bash
git init
git add main.go Dockerfile
git commit -m "Add Go API and multi-stage Dockerfile"
git branch -M main
git remote add origin https://github.com/<user>/go-api-gitops.git
git push -u origin main
```

---

## Phase 2 — Config repo + Argo CD Application

Clone the config repo and add three files.

**`manifests/deployment.yaml`:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: go-api
  labels:
    app: go-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: go-api
  template:
    metadata:
      labels:
        app: go-api
    spec:
      containers:
        - name: go-api
          image: syeddocker04/go-api:v1
          ports:
            - containerPort: 8080
```

The `image:` line is the one the pipeline rewrites on every push. `selector.matchLabels` must equal `template.metadata.labels` (`app: go-api`). No `namespace:` field — the Application decides that.

**`manifests/service.yaml`:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: go-api
spec:
  type: NodePort
  selector:
    app: go-api
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30950
```

**`application.yaml`** — the Argo CD Application, defined as version-controlled YAML instead of clicking the UI:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: go-api
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<user>/go-api-config.git
    targetRevision: HEAD
    path: manifests
  destination:
    server: https://kubernetes.default.svc
    namespace: go-api
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

**Why define the Application as YAML:** with the UI, the Application config lives only inside the cluster — if the cluster dies, it's gone and must be re-clicked from memory. As a Git file, the config *survives the cluster*: rebuild the cluster, reinstall Argo CD, run one `kubectl apply -f application.yaml`, and the Application is back identically. It also gains version history, review, and reproducibility across environments.

- `metadata.namespace: argocd` = where the Application object itself lives (where Argo CD looks for it).
- `destination.namespace: go-api` = where the *app* deploys (different namespace, different job).
- `syncPolicy.automated` = auto-sync + prune + self-heal, all declared up front.

Validate, commit, apply:

```bash
kubectl apply --dry-run=client -f manifests/deployment.yaml
kubectl apply --dry-run=client -f manifests/service.yaml

git add manifests/ application.yaml
git commit -m "Add go-api manifests and Argo CD Application"
git push -u origin main

kubectl apply -f application.yaml     # creates the Application; Argo CD auto-deploys
```

Verify:

```bash
kubectl get application -n argocd     # Synced / Healthy
kubectl get pods -n go-api            # 2 pods Running
# browse http://localhost -> "Version 1"
```

> `kubectl apply -f application.yaml` creates **one object** (the Application). Argo CD then does the actual deploying (reads the repo, applies manifests, creates the `go-api` namespace, starts pods).

---

## Phase 3 — GitHub Actions CI

### Secrets (never put credentials in the workflow file)

The workflow is committed to a public repo, so credentials live as **encrypted GitHub secrets**, injected at runtime and masked in logs. Add three to the **`go-api-gitops`** repo (Settings -> Secrets and variables -> Actions):

| Secret | Value |
|--------|-------|
| `DOCKERHUB_USERNAME` | your Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token (Read & Write) |
| `CONFIG_REPO_TOKEN` | GitHub classic PAT with `repo` scope (lets CI write to the config repo) |

`CONFIG_REPO_TOKEN` is needed because the pipeline runs in the app repo but must commit to the *config* repo — cross-repo write access.

### The workflow

**`.github/workflows/ci.yaml`** (in `go-api-gitops`):

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-update:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout app code
        uses: actions/checkout@v4

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: syeddocker04/go-api:${{ github.sha }}

      - name: Update image tag in config repo
        run: |
          git clone https://x-access-token:${{ secrets.CONFIG_REPO_TOKEN }}@github.com/<user>/go-api-config.git
          cd go-api-config
          sed -i "s|image: syeddocker04/go-api:.*|image: syeddocker04/go-api:${{ github.sha }}|" manifests/deployment.yaml
          git config user.name "github-actions"
          git config user.email "actions@github.com"
          git commit -am "Update image tag to ${{ github.sha }}"
          git push
```

**Line-by-line:**
- `on: push: branches: [main]` — the auto-run trigger. A push to main fires the workflow; you never run it manually.
- `actions/checkout@v4` — clones the app repo onto the runner (gives the build `main.go` + `Dockerfile`).
- `docker/login-action@v3` — authenticates to Docker Hub using the secrets (referenced, never printed).
- `docker/build-push-action@v5` — runs `docker build` (using the existing Dockerfile) and pushes. Tagged with `github.sha` (the commit hash) so every image traces to its exact source commit.
- **The bridge step** — clones the *config* repo (using `CONFIG_REPO_TOKEN`), `sed`-replaces the `image:` tag with the new SHA, commits, and pushes. This commit is what Argo CD reacts to.

Push the workflow — which itself triggers the first run:

```bash
git add .github/workflows/ci.yaml
git commit -m "Add CI pipeline: build, push, update config repo"
git push origin main
```

---

## Phase 4 — The finale: code change -> auto-deploy

Change one line and push nothing else:

```bash
sed -i 's/Version 1/Version 2/' main.go
git commit -am "Update greeting to Version 2"
git push origin main
```

Then watch — no other action:

```bash
kubectl get pods -n go-api -w
```

The chain runs itself: Actions builds a new SHA-tagged image -> pushes to Docker Hub -> commits the new tag to the config repo -> Argo CD syncs -> old pods `Terminating`, new pods `Running`. Refresh **http://localhost** -> **"Version 2"**.

---

## Tests / Demos

### Test 1 — CI ran automatically on push
GitHub -> `go-api-gitops` -> **Actions** tab. The run appears, triggered by the push, and goes green. No manual trigger.

### Test 2 — a uniquely-tagged image reached Docker Hub
`hub.docker.com/r/<user>/go-api` -> **Tags**. A new tag equal to the commit SHA appears. Traceable: the image maps to the exact commit that built it.

### Test 3 — the config repo was auto-updated (the bridge)
GitHub -> `go-api-config` -> history of `manifests/deployment.yaml`. A commit **"Update image tag to <sha>"** authored by `github-actions`, touching only that file. This is CI writing to a *different* repo.

### Test 4 — Argo CD deployed the new image
```bash
kubectl get deployment go-api -n go-api -o jsonpath='{.spec.template.spec.containers[0].image}' && echo
```
Ends in the new SHA, not `:v1`. Argo CD saw the config change and rolled it out.

### Test 5 — end to end from a code change
Edit `main.go`, push, do nothing else -> browser shows the new version. Zero `docker`, zero `kubectl`, zero manual deploy.

### Test 6 — disaster recovery (Application-as-YAML)
```bash
kind delete cluster --name gitops-demo
kind create cluster --name gitops-demo --config kind-config.yaml
kubectl create namespace argocd
kubectl apply -n argocd --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl apply -f application.yaml     # one command restores the whole app
```
Because the Application and manifests live in Git, the entire deployment rebuilds from one `kubectl apply`. The cluster is disposable; Git is the system.

---

## Cleanup

```bash
kind delete cluster --name gitops-demo     # repos on GitHub/Docker Hub are untouched
```

---

## Lessons learned

- **CI is not CD.** GitHub Actions builds and publishes; Argo CD deploys. CI never touches the cluster and holds no cluster credentials — a real security win.
- **Two repos break the trigger loop** and separate *what the app is* from *what runs*. CI (in the app repo) writes the image tag to the config repo; Argo CD watches only the config repo.
- **Argo CD watches Git, not the registry.** A new image alone does nothing; the manifest tag must change. That tag-update commit is the bridge.
- **Tag by commit SHA** for traceability — a running pod maps to an exact source commit.
- **Secrets live in a secret store, never in code.** Three tokens, referenced by name, masked in logs.
- **Multi-stage Docker** shrinks the image from ~800MB to ~8MB by shipping only the static binary.
- **Application-as-YAML** makes the deploy config survive the cluster: rebuild + one `kubectl apply` restores everything.
- **Rolling updates** bring new pods up healthy before old ones terminate; brief `Error`/`Terminating` on old pods during rollout is normal churn, not failure.

---

## Tech stack

Go · Docker (multi-stage) · Docker Hub · kind · Kubernetes · GitHub Actions · Argo CD

---



- **App-of-Apps**: a root Application that watches a repo of Application YAMLs, so even the Applications are reconciled from Git (fully recursive GitOps).
- **Argo CD Image Updater**: let a controller watch the registry and bump tags, removing the CI `sed` step.
- **Multi-environment**: dev/staging/prod via Kustomize overlays, promoted through Git.
