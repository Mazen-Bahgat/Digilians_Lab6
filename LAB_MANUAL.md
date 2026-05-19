# CISC 814 — Lab 6

## Kubernetes Orchestration, GitOps with Argo CD, and Observability with Grafana

---

## 1. Learning Objectives

By the end of this lab you will be able to:

1. **Provision** a multi-node Kubernetes cluster locally using k3d, and identify the same components on a Docker Desktop Kubernetes cluster.
2. **Author** raw Kubernetes manifests for `Deployment`, `Service`, `ConfigMap`, and `Namespace`, and explain how `labels` and `selectors` wire them together.
3. **Refactor** flat manifests into a `Kustomize` base with `dev` and `prod` overlays, applying replica counts and configuration patches per environment.
4. **Install** Argo CD into your cluster and configure it to reconcile applications from a git repository (pull-based GitOps).
5. **Demonstrate** drift detection and automated self-healing by mutating cluster state out-of-band and observing Argo CD's response.
6. **Deploy** Prometheus and Grafana via Helm, scrape pod-level metrics, and build a dashboard that surfaces pod health, CPU, memory, and restart counts.
7. **Reason** about how the techniques in this lab map to real production engineering decisions through four short corporate scenarios.

---

### 2.1 Kubernetes (the orchestrator)

Kubernetes runs containerised workloads across a fleet of machines (nodes). You declare the _desired_ state in YAML manifests; the cluster's control plane works continuously to make the _actual_ state match. The unit of work is the **Pod** (one or more co-located containers). Pods are normally created in groups by a **Deployment**, which manages rollouts, rollbacks, and replica counts. A **Service** gives that group of Pods a stable virtual IP and DNS name, because individual Pods are ephemeral.

**Labels** (key/value tags on objects) and **selectors** (queries that match labels) are how Kubernetes objects find each other. A Service does not know which Pods it routes to by name — it knows by label selector. Misaligned labels are the single most common reason a Service appears to "have no endpoints."

**Namespaces** are soft partitions of the cluster. They scope object names, RBAC, network policies, and resource quotas. We will use `sample-web-dev`, `sample-web-prod`, `podinfo-dev`, `podinfo-prod`, `argocd`, `ingress-nginx`, and `monitoring` in this lab.

### 2.2 GitOps with Argo CD (the reconciler)

A **GitOps** workflow treats a git repository as the single source of truth for what should be running in the cluster. Instead of `kubectl apply`-ing from a CI pipeline (push model), an agent inside the cluster (Argo CD, in our case) continuously _pulls_ the desired state from git and reconciles the cluster to match (pull model).

This buys you two important properties:

- **Drift detection** — if someone hand-edits the cluster (`kubectl edit deployment …`), Argo CD reports the live state as "OutOfSync."
- **Self-healing** — with `selfHeal: true` enabled, Argo CD automatically reverts the cluster to the git-declared state, eliminating configuration drift.

Argo CD's central object is the `Application` Custom Resource. Each Application points at a path inside a git repo and a destination namespace in the cluster, and chooses a tool to render manifests (we use **Kustomize**).

### 2.3 Observability with Prometheus + Grafana (the eyes)

**Prometheus** is a pull-based metrics database. It periodically scrapes HTTP `/metrics` endpoints exposed by your applications and by Kubernetes components (kubelet, kube-state-metrics, node-exporter), stores the resulting time series, and lets you query them with PromQL.

**Grafana** is a visualisation layer that talks to Prometheus (and other backends) and renders dashboards. Standard dashboards include cluster health, pod CPU and memory usage, pod restart counts, deployment rollout status, and per-namespace resource consumption.

In production you would add log aggregation (Loki, ELK) and distributed tracing (Tempo, Jaeger). Those are out of scope here — the goal is to make sure you can install the metrics stack, point it at your apps, and read a dashboard.

### 2.4 Kustomize (template-free configuration management)

`Kustomize` lets you keep a single **base** of plain Kubernetes YAML and produce environment-specific variants by layering **overlays** on top. Overlays are themselves YAML — they describe patches, replica overrides, image tag changes, name prefixes, and namespace targeting. Unlike Helm, Kustomize has no templating language; you compose YAML with YAML. It is built into `kubectl` (`kubectl apply -k <path>`) and is the native tool Argo CD uses to render this lab's apps.

---

## 3. Pre-Lab Setup

Complete every item in this section **before** attempting the procedure. Allow 30–45 minutes if you are starting from a fresh laptop. **All commands in this manual use bash** — if you are on Windows, use WSL (Ubuntu 22.04 or later).

### 3.1 System requirements

| Resource  | Minimum                                                   | Recommended |
| --------- | --------------------------------------------------------- | ----------- |
| RAM       | 8 GB                                                      | 16 GB       |
| Free disk | 10 GB                                                     | 20 GB       |
| CPU       | 4 cores                                                   | 8 cores     |
| Network   | Reliable internet for image pulls                         | —           |
| OS        | Linux (Ubuntu 22.04+), Windows + WSL 2 (Ubuntu), or macOS | —           |

### 3.2 Required tools

You will install five tools. Versions shown have been validated for this lab.

| Tool      | Purpose                             | Tested version |
| --------- | ----------------------------------- | -------------- |
| `docker`  | Container runtime (required by k3d) | 27.x+          |
| `k3d`     | Lightweight Kubernetes-in-Docker    | 5.7+           |
| `kubectl` | Kubernetes CLI                      | 1.30+          |
| `helm`    | Kubernetes package manager          | 3.15+          |
| `git`     | Source control for GitOps           | 2.40+          |

### 3.3 Install everything (Ubuntu / WSL)

```bash
# Docker Engine
# verify docker

# kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -m 0755 kubectl /usr/local/bin/kubectl
rm kubectl

# Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# k3d
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

```

### 3.4 Install everything (macOS, with Homebrew)

```bash
brew install --cask docker
open -a Docker                       # launch Docker Desktop once
brew install kubectl helm k3d git
```

### 3.5 Tool verification

Run each command and confirm it prints a version (exact numbers may vary):

```bash
docker --version
kubectl version --client
helm version
git --version
k3d version
```

### 3.6 Clone or fork this repository

```bash
# In the directory where you keep coursework:
git clone https://github.com/TasnimElyamany/Digilians_Lab6.git
cd Digilians_Lab6
```

### 3.7 Rewrite the Argo CD `repoURL` placeholders

The four Argo CD `Application` manifests (and the root `apps.yaml`) ship with the placeholder URL `https://github.com/REPLACE-ME/cisc814-k8s-lab.git`. Argo CD cannot pull from a placeholder, so you must rewrite these to point at your own fork **before** Part 4.

Confirm which files still contain the placeholder:

```bash
grep -rl 'REPLACE-ME' --include='*.yaml' .
```

Rewrite all of them in one shot (substitute your own fork URL):

```bash
YOUR="https://github.com/TasnimElyamany/Digilians_Lab6.git"
find . -name '*.yaml' -type f -exec \
  sed -i "s|https://github.com/REPLACE-ME/cisc814-k8s-lab.git|$YOUR|g" {} +
```

Confirm no placeholders remain:

```bash
grep -r 'REPLACE-ME' --include='*.yaml' . && echo "FOUND" || echo "clean"
```

(`clean` means success.) Commit and push:

```bash
git add .
git commit -m "Point Argo CD apps at my fork"
git push
```

> **Why this is necessary.** Argo CD runs _inside_ the cluster and pulls manifests over HTTPS. Even though you are running on your laptop, Argo CD cannot read your local filesystem — it can only read what you have pushed to a public git host.

---

## 4. Procedure

### Part 1 — Create a Local Kubernetes Cluster

**Goal.** End this part with a running cluster, a configured `kubectl` context, and visibility into nodes and system pods.

You have two cluster options. Pick **one** and stick with it for the rest of the lab.

#### 1.A — k3d (recommended)

k3d wraps the lightweight K3s distribution in Docker containers. It is fast (~30 s to create), supports multiple nodes, and tears down cleanly. The lab assumes k3d unless noted.

**First, confirm Docker is running:**

```bash
docker info | grep -i "server version"
```

If this errors, start Docker Desktop (or `sudo service docker start` on native Linux) and wait until the command succeeds.

**Create the cluster:**

```bash
k3d cluster create cisc814 \
  --servers 1 \
  --agents 2 \
  --port "8080:8080@loadbalancer" \
  --port "8443:8443@loadbalancer" \
  --k3s-arg "--disable=traefik@server:*" \
  --wait
```

**Explanation of each flag:**

| Flag                                     | Purpose                                                                                                                                                                  |
| ---------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `cisc814`                                | The cluster name. `kubectl` context will be set to `k3d-cisc814`.                                                                                                        |
| `--servers 1 --agents 2`                 | One control-plane node and two worker nodes — enough to demonstrate pod scheduling and node selectors.                                                                   |
| `--port "8080:8080@loadbalancer"`        | Maps host port 8080 to the cluster's load balancer port 8080. If you install ingress-nginx in Part 2.5, this is how `http://*.localtest.me:8080` reaches the controller. |
| `--port "8443:8443@loadbalancer"`        | Same for HTTPS (port 8443) — used by ingress-nginx in Part 2.5.                                                                                                          |
| `--k3s-arg "--disable=traefik@server:*"` | We do not need K3s's bundled Traefik ingress controller for this lab; disabling it saves ~150 MB of RAM (we install our own ingress-nginx in Part 2.5).                  |
| `--wait`                                 | Block until the cluster is fully ready before returning.                                                                                                                 |

**Set the kubectl context and verify:**

```bash
kubectl config use-context k3d-cisc814
kubectl cluster-info
kubectl get nodes
```

Expected: one `control-plane` node and two `worker` nodes, all in `Ready` state.

#### 1.B — Docker Desktop Kubernetes (secondary)

If k3d will not run (uncommon but possible on locked-down corporate laptops), enable Docker Desktop's built-in single-node Kubernetes:

1. Open **Docker Desktop → Settings → Kubernetes**.
2. Check **Enable Kubernetes** and click **Apply & Restart**.
3. Wait for the Kubernetes status indicator at the bottom-left to turn green (~2 minutes on first enable).

Then switch the kubectl context:

```bash
kubectl config use-context docker-desktop
kubectl cluster-info
kubectl get nodes
```

You will see a single node — for this lab that is acceptable, but scheduling demos that rely on multiple nodes (e.g. anti-affinity) will be less interesting.

> **The rest of this lab assumes you are on k3d.** Where Docker Desktop behaves differently, the difference is called out inline.

#### 1.1 — Inspect what you got

```bash
kubectl get nodes -o wide
kubectl get namespaces
kubectl get pods --all-namespaces
```

You should see the `kube-system` namespace populated with `coredns`, `local-path-provisioner`, `metrics-server`, and (on k3d) `svclb-*` pods.

---

### Part 2 — Kubernetes Basics: Manual Deployment

**Goal.** Deploy the `sample-web` app using raw `kubectl apply -f` against the **base** manifests, then dissect what was created.

#### 2.1 — Read the manifests before applying them

Open and skim:

- [apps/sample-web/base/namespace.yaml](apps/sample-web/base/namespace.yaml)
- [apps/sample-web/base/configmap.yaml](apps/sample-web/base/configmap.yaml)
- [apps/sample-web/base/deployment.yaml](apps/sample-web/base/deployment.yaml)
- [apps/sample-web/base/service.yaml](apps/sample-web/base/service.yaml)
- [apps/sample-web/base/ingress.yaml](apps/sample-web/base/ingress.yaml) — used only if you install ingress-nginx in Part 2.5

Note three things:

- The `Deployment.spec.selector.matchLabels` and the `Service.spec.selector` **must match** the labels on the Pod template (`spec.template.metadata.labels`). If you change one, change all three.
- The `Deployment` mounts the `ConfigMap` as a volume into the nginx HTML directory, so editing the ConfigMap will change what the website serves.
- Everything is namespaced to `sample-web` (the namespace defined in `namespace.yaml`).

#### 2.2 — Apply the base manually

We are deliberately **not** using Kustomize in this part — that comes in Part 3. So we apply each base manifest by name, skipping `kustomization.yaml` (which is a Kustomize control file, not a Kubernetes resource).

```bash
kubectl apply -f apps/sample-web/base/namespace.yaml

kubectl apply -n sample-web \
  -f apps/sample-web/base/configmap.yaml \
  -f apps/sample-web/base/deployment.yaml \
  -f apps/sample-web/base/service.yaml \
  -f apps/sample-web/base/ingress.yaml
```

Expected output (order may vary):

```text
namespace/sample-web created
configmap/sample-web-content created
deployment.apps/sample-web created
service/sample-web created
ingress.networking.k8s.io/sample-web created
```

> **Why not `kubectl apply -f apps/sample-web/base/`?** That shorthand would also try to apply `kustomization.yaml`, which is not a Kubernetes resource — `kubectl` would print `no matches for kind "Kustomization"` at the end. The actual manifests still get applied (so it is not fatal), but listing files explicitly here keeps the Part 2 / Part 3 separation clean.
>
> The Ingress will appear with `ADDRESS` empty until you install ingress-nginx in Part 2.5. That is harmless — the Service and port-forward still work without it.

#### 2.3 — Watch the rollout

```bash
kubectl get deployment sample-web -n sample-web
kubectl get pods -n sample-web -w        # Ctrl+C when all Pods are Running
```

#### 2.4 — Inspect labels and selectors

```bash
kubectl get pods -n sample-web --show-labels
kubectl get endpoints sample-web -n sample-web
kubectl describe service sample-web -n sample-web
```

The `Endpoints` resource lists the Pod IPs that the Service is currently routing to. Confirm the count of endpoint IPs equals the count of Ready Pods. If the Endpoints list is empty, your selector and Pod labels disagree.

#### 2.5 — Reach the website

```bash
kubectl port-forward -n sample-web svc/sample-web 8081:80
```

Open `http://localhost:8081` in a browser. You should see the page **"CISC 814 — sample-web (base, no overlay)"**. Press Ctrl+C in the terminal to stop the port-forward.

#### 2.6 — Modify a label and break the Service (instructive)

Run:

```bash
kubectl label pod -n sample-web -l app=sample-web app=broken-on-purpose --overwrite
kubectl get endpoints sample-web -n sample-web
```

The Endpoints list drains to empty within a second — the Service can no longer find the Pods. Restore:

```bash
kubectl label pod -n sample-web -l app=broken-on-purpose app=sample-web --overwrite
```

This is the most common production K8s bug. Internalise it now.

#### 2.7 — Tear down before moving to Kustomize

```bash
kubectl delete namespace sample-web
```

---

### Part 2.5 — Install ingress-nginx (optional but recommended)

**Goal.** Replace `kubectl port-forward` with a real ingress controller. After this part you will reach apps in your browser at human-readable URLs like `http://sample-web-dev.localtest.me:8080`, which is how production clusters work.

**Why "optional but recommended."** The rest of the lab functions perfectly without it — every Service is also reachable via `kubectl port-forward`. But Ingress is industry-standard, and the `Ingress` resources are already declared in the manifests. Installing the controller takes ~3 minutes and makes the GitOps and monitoring URLs much nicer.

#### 2.5.1 — Why `localtest.me`

You need DNS that resolves to your cluster's load balancer (which is itself bound to `127.0.0.1` on your laptop). The public service `localtest.me` resolves **every** subdomain to `127.0.0.1`. So `sample-web-dev.localtest.me`, `podinfo-prod.localtest.me`, and any other `*.localtest.me` all resolve to localhost — no `/etc/hosts` edits required. The hostname is what the Ingress controller uses to route to the right backend Service.

#### 2.5.2 — Install ingress-nginx via Helm

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx --create-namespace \
  -f infrastructure/networking/ingress-nginx/values.yaml

kubectl -n ingress-nginx rollout status deploy/ingress-nginx-controller --timeout=180s
kubectl -n ingress-nginx get svc ingress-nginx-controller
```

The values file ([infrastructure/networking/ingress-nginx/values.yaml](infrastructure/networking/ingress-nginx/values.yaml)) configures:

- `controller.service.type: LoadBalancer` with `ports.http: 8080` and `ports.https: 8443` — these match the host port mappings you configured when you created the k3d cluster (or that Docker Desktop publishes automatically).
- `controller.admissionWebhooks.enabled: false` — the webhook is occasionally flaky on slow laptops; it's a defence-in-depth feature you do not need in a lab.

**Verify the controller is reachable.** From any terminal:

```bash
curl http://localhost:8080
```

You should see a 404 page served by nginx itself — this is **correct** at this stage and proves the controller is up. A 404 from nginx means "I'm here but no Ingress rule matches the Host header you sent" — which is true, because you haven't asked for a real hostname yet.

#### 2.5.3 — Verify the Ingress resources

The Kustomize bases for both `sample-web` and `podinfo` already include `ingress.yaml`. Once Argo CD has synced (Part 4), four Ingress resources will appear automatically:

```bash
kubectl get ingress --all-namespaces
```

Expected after Part 4:

```text
NAMESPACE          NAME         CLASS   HOSTS                              ADDRESS     PORTS   AGE
sample-web-dev     sample-web   nginx   sample-web-dev.localtest.me        localhost   80      2m
sample-web-prod    sample-web   nginx   sample-web-prod.localtest.me       localhost   80      2m
podinfo-dev        podinfo      nginx   podinfo-dev.localtest.me           localhost   80      2m
podinfo-prod       podinfo      nginx   podinfo-prod.localtest.me          localhost   80      2m
```

If you want to apply the Ingress now (before Part 4 / Argo CD), `kubectl apply -k apps/sample-web/overlays/dev` will create it along with everything else.

#### 2.5.4 — Browse the apps

Open in any browser:

| URL                                        | What you should see                   |
| ------------------------------------------ | ------------------------------------- |
| `http://sample-web-dev.localtest.me:8080`  | Blue "DEV environment" banner         |
| `http://sample-web-prod.localtest.me:8080` | Orange "PROD environment" banner      |
| `http://podinfo-dev.localtest.me:8080`     | podinfo UI showing pod name + version |
| `http://podinfo-prod.localtest.me:8080`    | Same UI, different pod count          |

If you get **"connection refused,"** ingress-nginx is not bound to `localhost:8080` — check `kubectl get svc -n ingress-nginx` and confirm the `EXTERNAL-IP` is `localhost` and the port is `8080`.

If you get **"404 Not Found"** from nginx, the Host header is not matching any Ingress rule — confirm `kubectl get ingress -A` shows your host, and that you typed the URL exactly (the trailing port `:8080` matters).

#### 2.5.5 — Note for Argo CD's port-forward

ingress-nginx binds `localhost:8443` (HTTPS) on Docker Desktop. To avoid colliding with that, the Argo CD UI port-forward in Part 4 uses port **9443** instead of 8443. The k3d path is identical (we use 9443 there too for consistency).

> **Skip this part?** Every subsequent command in the manual still works via `kubectl port-forward`. You will just substitute `http://localhost:8081` (and friends) for the `*.localtest.me:8080` URLs.

---

### Part 3 — Kustomize Base and Overlays

**Goal.** Apply the same `sample-web` app twice — once as `dev` (1 replica, dev marker page) and once as `prod` (3 replicas, prod marker page) — using overlays on top of an unchanged base.

#### 3.1 — Read the overlay structure

- Base: [apps/sample-web/base/kustomization.yaml](apps/sample-web/base/kustomization.yaml)
- Dev overlay: [apps/sample-web/overlays/dev/kustomization.yaml](apps/sample-web/overlays/dev/kustomization.yaml)
- Prod overlay: [apps/sample-web/overlays/prod/kustomization.yaml](apps/sample-web/overlays/prod/kustomization.yaml)

The dev overlay sets `replicas: 1`, namespace `sample-web-dev`, patches the ConfigMap text, and rewrites the Ingress host to `sample-web-dev.localtest.me`. Prod sets `replicas: 3`, namespace `sample-web-prod`, a different ConfigMap text, and the host `sample-web-prod.localtest.me`.

#### 3.2 — Render without applying (always do this first)

```bash
kubectl kustomize apps/sample-web/overlays/dev
kubectl kustomize apps/sample-web/overlays/prod
```

This prints the final composed YAML to stdout. Confirm namespaces, replica counts, ConfigMap content, and Ingress hosts reflect each environment.

#### 3.3 — Apply both overlays

```bash
kubectl apply -k apps/sample-web/overlays/dev
kubectl apply -k apps/sample-web/overlays/prod
```

Verify:

```bash
kubectl get deploy,svc,ingress,pods -n sample-web-dev
kubectl get deploy,svc,ingress,pods -n sample-web-prod
```

Expected: 1 Pod in dev, 3 Pods in prod, each with its own Service and Ingress.

#### 3.4 — Test both environments

If you installed ingress-nginx in Part 2.5, just open the browser:

- `http://sample-web-dev.localtest.me:8080`
- `http://sample-web-prod.localtest.me:8080`

If you skipped Part 2.5, use port-forward in two terminals:

```bash
kubectl port-forward -n sample-web-dev svc/sample-web 8081:80    # Ctrl+C to stop
kubectl port-forward -n sample-web-prod svc/sample-web 8082:80   # in a new terminal, Ctrl+C to stop
```

Open `http://localhost:8081` (dev banner) and `http://localhost:8082` (prod banner).

#### 3.5 — Teardown before Argo CD

```bash
kubectl delete -k apps/sample-web/overlays/dev
kubectl delete -k apps/sample-web/overlays/prod
```

You are now going to let Argo CD apply these same overlays _for_ you — the manual run was so you can see what Argo CD will do later.

---

### Part 4 — GitOps with Argo CD

**Goal.** Install Argo CD, register the four Applications (`sample-web-dev`, `sample-web-prod`, `podinfo-dev`, `podinfo-prod`), demonstrate drift detection, and enable self-healing.

> **Before you start:** complete §3.6 and §3.7 — Argo CD must be able to reach your git fork over the public internet, and the `repoURL` placeholders must be rewritten to your fork.

#### 4.1 — Install Argo CD

```bash
kubectl create namespace argocd

kubectl apply -n argocd \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.13.0/manifests/install.yaml

kubectl -n argocd rollout status deploy/argocd-server --timeout=300s
```

**Retrieve the initial admin password:**

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d ; echo
```

Copy the password — you will use it in §4.2 to log in as `admin`.

#### 4.2 — Open the Argo CD UI

```bash
kubectl port-forward -n argocd svc/argocd-server 9443:443
```

Open `https://localhost:9443` (your browser will warn about the self-signed certificate — click through). Log in as `admin` with the password from §4.1. You will see an empty Applications list.

> **Why port 9443 and not 8443?** If you completed Part 2.5, ingress-nginx is already bound to `localhost:8443` for HTTPS. Using 9443 avoids the collision. If you skipped Part 2.5, you can use 8443 freely.

#### 4.3 — Bootstrap the four Applications

Open a **new** terminal (leave the port-forward running) and apply the single "app-of-apps" root Application:

```bash
kubectl apply -n argocd -f clusters/local/apps.yaml
```

That root Application ([clusters/local/apps.yaml](clusters/local/apps.yaml)) points Argo CD at the [infrastructure/argocd/applications/apps/](infrastructure/argocd/applications/apps/) directory, where four child Application manifests live. Within ~30 seconds Argo CD discovers and creates all four child applications.

Confirm in the UI: you should see five tiles (`apps`, `sample-web-dev`, `sample-web-prod`, `podinfo-dev`, `podinfo-prod`), each progressing through `Missing → OutOfSync → Synced` and `Progressing → Healthy`.

From the CLI:

```bash
kubectl get applications -n argocd
kubectl get pods --all-namespaces | grep -E "sample-web|podinfo"
```

> **If everything is stuck `Unknown` or `ComparisonError`:** the most likely cause is §3.7 was skipped or incomplete. Re-run `grep -r 'REPLACE-ME' --include='*.yaml' .` — if anything matches, run the `sed` rewrite again, then in the Argo CD UI click each app → **Refresh**.

#### 4.4 — Demonstrate drift detection

With the dev app `Synced`, manually mutate the live state:

```bash
kubectl scale deployment sample-web -n sample-web-dev --replicas=5
```

Within ~3 minutes (the default refresh interval) Argo CD will report `sample-web-dev` as `OutOfSync`. You can force an immediate refresh in the UI by clicking the Application and pressing **Refresh**.

Click the Application tile and look at the diff — it will show `replicas: 5` (live) versus `replicas: 1` (desired). This is **drift detection**.

#### 4.5 — Demonstrate self-healing

By default the child Applications in this lab have:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

That means Argo CD does not just detect drift — it reverts it. Confirm:

```bash
kubectl get deployment sample-web -n sample-web-dev -w
```

Within seconds of the `selfHeal` controller cycling, you will see replicas drop from 5 back to 1. Press Ctrl+C.

Repeat the drift attack with `kubectl edit`, `kubectl patch`, or `kubectl delete pod` — Argo CD restores every change.

#### 4.6 — Make a real change the GitOps way

Edit [apps/sample-web/overlays/dev/replicas-patch.yaml](apps/sample-web/overlays/dev/replicas-patch.yaml) and change replicas from `1` to `2`. Commit and push:

```bash
git add apps/sample-web/overlays/dev/replicas-patch.yaml
git commit -m "Scale dev to 2 replicas"
git push
```

Within ~3 minutes (or immediately on a Refresh) Argo CD will sync and the dev deployment will scale to 2.

> **Key insight.** The cluster trusts only git. Cluster mutations made by humans are treated as drift. To change production, you change a file, open a PR, merge, and Argo CD applies — the same workflow you already use for code.

---

### Part 5 — Observability with Prometheus and Grafana

**Goal.** Install Prometheus and Grafana into a dedicated `monitoring` namespace, scrape `podinfo`'s `/metrics` endpoint, and read a dashboard showing pod health and resource usage.

#### 5.1 — Install metrics-server (if not present)

k3d ships with metrics-server. Docker Desktop usually does not. Check:

```bash
kubectl get deployment metrics-server -n kube-system
```

If missing, install it:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# For Docker Desktop / kind only, patch to skip TLS verification (NOT for production):
kubectl patch deployment metrics-server -n kube-system --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
```

Test:

```bash
kubectl top nodes
kubectl top pods --all-namespaces
```

#### 5.2 — Install Prometheus

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update

kubectl create namespace monitoring

helm install prometheus prometheus-community/prometheus \
  -n monitoring \
  -f infrastructure/observability/prometheus/values.yaml
```

The values file ([infrastructure/observability/prometheus/values.yaml](infrastructure/observability/prometheus/values.yaml)) keeps resource requests modest for a laptop and ensures the default `kubernetes-pods` scrape job is enabled. That job looks for the annotation `prometheus.io/scrape: "true"` on Pods — which is exactly what we set on `podinfo`.

Wait for readiness:

```bash
kubectl -n monitoring rollout status deploy/prometheus-server --timeout=300s
```

#### 5.3 — Install Grafana

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install grafana grafana/grafana \
  -n monitoring \
  -f infrastructure/observability/grafana/values.yaml

kubectl -n monitoring rollout status deploy/grafana --timeout=300s
```

The values file ([infrastructure/observability/grafana/values.yaml](infrastructure/observability/grafana/values.yaml)) preconfigures the Prometheus data source and auto-imports three community dashboards by ID.

**Retrieve the admin password:**

```bash
kubectl get secret -n monitoring grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

#### 5.4 — Open Grafana

```bash
kubectl port-forward -n monitoring svc/grafana 3000:80
```

Open `http://localhost:3000`. Log in as `admin` with the password from §5.3.

Navigate to **Dashboards → Browse → CISC 814**. You will see three pre-imported dashboards:

- **Kubernetes / Compute Resources / Pod** (ID 6336) — per-pod CPU, memory, network, and restart counts.
- **Kubernetes / Views / Pods** (ID 15760) — overview of all pods with status badges.
- **Node Exporter Full** (ID 1860) — node-level CPU, memory, disk, and network.

#### 5.5 — Query podinfo's custom metrics

In **Explore → Prometheus**, paste:

```promql
sum(rate(http_requests_total{namespace="podinfo-dev"}[2m]))
```

You should see a flat zero line — podinfo is up but nobody is hitting it. Generate some traffic:

```bash
# Terminal A:
kubectl port-forward -n podinfo-dev svc/podinfo 9898:9898

# Terminal B:
for i in $(seq 1 100); do curl -s http://localhost:9898/ > /dev/null; sleep 0.2; done
```

Re-run the PromQL query. The line should rise to ~5 req/s.

#### 5.6 — Build your own panel

In Grafana, **Create → Dashboard → Add visualization**. Pick `Prometheus` as the data source and paste:

```promql
sum by (pod) (kube_pod_container_status_restarts_total{namespace=~"sample-web-.*|podinfo-.*"})
```

Save the dashboard as `CISC 814 — My Pods`. You will capture a screenshot of this in §7.

---

### Part 6 — Corporate Scenarios

Read each scenario and answer in your lab report (1 short paragraph each). These are not trick questions — they test whether you understand _why_ the tooling you just used exists.

#### Scenario 1 — The 2 a.m. config edit

A senior engineer at your company logs into the production cluster at 2 a.m. to "quickly bump replicas" on a struggling service. They forget to update the git repo. The next morning, an automated promotion from staging to production overwrites their change and the incident recurs.

> Using the concepts from Part 4, explain why this happened and how Argo CD's `selfHeal: true` and a culture of "no manual cluster edits" prevent it.

#### Scenario 2 — The dev/prod parity drift

A team manages dev and prod with two separate, hand-maintained YAML directories. Over six months, dev's nginx version is updated three times but prod is not. A vulnerability disclosure forces an emergency prod upgrade, which immediately fails with a config incompatibility nobody saw coming.

> Using the concepts from Part 3, explain how a Kustomize `base` + overlays structure would have prevented this and what concrete change you would make to the team's repository.

#### Scenario 3 — The mystery pod restart

A backend pod is restarting every ~12 minutes in production. The team has no alerting and only finds out when a customer complains. Logs are gone by the time anyone investigates because the pod has restarted.

> Using the concepts from Part 5, list three Prometheus/Grafana signals (with PromQL queries if you can write them) that would have surfaced this within minutes, and explain why a log-only solution is not enough.

#### Scenario 4 — The compliance auditor

An auditor asks: "Show me every change made to the production cluster in the last 90 days, who approved it, and what the cluster looked like on day 47."

> Using the concepts from Part 4, explain how a GitOps workflow answers this question essentially for free, and what a push-based CI/CD workflow would have to do to produce the same evidence.

---

## 5. Exercises

You will be graded on the artefacts produced by these exercises (see §7).

### Exercise 1 — Manual deployment

Apply [apps/sample-web/base/](apps/sample-web/base/) using raw `kubectl apply -f`. Capture the output of:

```bash
kubectl get deploy,svc,ingress,pods,cm -n sample-web -o wide
```

### Exercise 2 — Kustomize overlays

Apply both `dev` and `prod` overlays for `sample-web`. Then write a short table in your report listing **every field that differs** between the two rendered manifests (use `kubectl kustomize` to render).

### Exercise 3 — GitOps deployment

After completing §4.3, screenshot the Argo CD UI showing all four child Applications as `Synced` and `Healthy`. The screenshot must include the sidebar so the cluster name is visible.

### Exercise 4 — Drift detection and self-healing

Perform the following drift attacks and document the outcome of each:

```bash
kubectl scale deployment sample-web -n sample-web-dev --replicas=7
kubectl delete pod -n sample-web-dev -l app=sample-web
kubectl set image deployment/sample-web -n sample-web-dev nginx=nginx:1.20-alpine
```

For each, record (i) the time Argo CD took to detect drift and (ii) the time it took to revert. Wall-clock times measured with `date` are fine.

### Exercise 5 — Grafana monitoring

Build the **CISC 814 — My Pods** dashboard described in §5.6 with at least three panels:

1. Pod restart count by pod name (5.6 query).
2. Pod CPU usage by pod, last 15 minutes.
3. Pod memory usage by pod, last 15 minutes.

Export the dashboard JSON (Dashboard → Share → Export → Save to file) and include it in your submission.

### Exercise 6 — Make a real GitOps change

Through git only (no `kubectl edit`!) increase the prod replicas for `podinfo` from 3 to 4. Commit, push, and capture the Argo CD UI showing the sync from `OutOfSync` to `Synced`.

---

## 6. Verification Checklist

Run each command below from the repo root after completing the full procedure. Tick the box if the output matches the expected description. **A complete tick list is part of the submission.**

| #   | Command                                                                                                                          | Expected                                                                                                      | ☐   |
| --- | -------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- | --- |
| 1   | `k3d cluster list`                                                                                                               | One cluster `cisc814`, running                                                                                | ☐   |
| 2   | `kubectl get nodes`                                                                                                              | 1 server + 2 agents (k3d) or 1 node (DD), all `Ready`                                                         | ☐   |
| 3   | `kubectl get ns`                                                                                                                 | Namespaces include `argocd`, `monitoring`, `sample-web-dev`, `sample-web-prod`, `podinfo-dev`, `podinfo-prod` | ☐   |
| 4   | `kubectl get applications -n argocd`                                                                                             | 5 apps (`apps`, four children), all `Synced` and `Healthy`                                                    | ☐   |
| 5   | `kubectl get deploy -n sample-web-dev`                                                                                           | `sample-web` with desired replicas matching the dev overlay                                                   | ☐   |
| 6   | `kubectl get deploy -n sample-web-prod`                                                                                          | `sample-web` with desired replicas matching the prod overlay                                                  | ☐   |
| 7   | `kubectl get deploy -n monitoring`                                                                                               | `prometheus-server` and `grafana` both `Available`                                                            | ☐   |
| 8   | `kubectl top pods --all-namespaces`                                                                                              | Returns metrics (not "metrics not available")                                                                 | ☐   |
| 9   | `kubectl get endpoints -n sample-web-prod sample-web`                                                                            | Endpoint list is non-empty                                                                                    | ☐   |
| 10  | Argo CD UI at `https://localhost:9443`                                                                                           | Login works, all apps green                                                                                   | ☐   |
| 10b | (if Part 2.5 done) `curl http://sample-web-dev.localtest.me:8080`                                                                | Returns the dev HTML                                                                                          | ☐   |
| 11  | Grafana UI at `http://localhost:3000`                                                                                            | Login works, Prometheus data source shows `Working`                                                           | ☐   |
| 12  | `kubectl scale deploy sample-web -n sample-web-dev --replicas=9` → wait 30 s → `kubectl get deploy sample-web -n sample-web-dev` | Replicas back to dev-overlay value (self-heal)                                                                | ☐   |

---

## 7. Submission Requirements

Submit a single `.zip` containing:

1. **`report.pdf`** — your lab report. Maximum 6 pages. Must include:
   - Cover page with name, student number, course, lab number, date.
   - Brief summary (≤ 200 words) of what you built.
   - Answers to all four scenarios in [Part 6](#part-6--corporate-scenarios).
   - Exercise 2 difference table.
   - Exercise 4 timing measurements table.
   - Verification checklist (§6) with all 12 boxes ticked or annotated.
   - **Screenshots:** Argo CD all-green UI (Ex. 3), Grafana custom dashboard (Ex. 5), Argo CD sync transition for podinfo prod (Ex. 6).
2. **`my-pods-dashboard.json`** — exported Grafana dashboard JSON from Exercise 5.
3. **`verification-output.txt`** — raw stdout from running each command in §6, top-to-bottom in one terminal session.
4. **`git-log.txt`** — output of `git log --oneline -- apps/ infrastructure/ clusters/`, showing your GitOps commit history (must include at least the commits from Exercises 6 and §4.6).

**Naming:** `<studentnumber>_lab6.zip`.

**Late policy:** as per course outline.

**Academic integrity:** the manifests in this repository are scaffolding — modifying overlays, writing your own dashboard, and authoring your report answers are individual work.

---

## 8. Marking Rubric

Total: **100 marks**.

| Section                          | Weight | Criteria                                                                                                                                                                           |
| -------------------------------- | ------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Pre-lab + cluster setup**      | 10     | Cluster running, tools verified, repo forked and `repoURL` rewritten.                                                                                                              |
| **Part 2 — K8s basics**          | 10     | Manifests applied; labels/selectors/endpoints discussion in report shows understanding (not just screenshots).                                                                     |
| **Part 3 — Kustomize overlays**  | 15     | Both overlays applied; Exercise 2 difference table is accurate and complete.                                                                                                       |
| **Part 4 — GitOps with Argo CD** | 25     | All four child applications Synced/Healthy; Exercise 3 screenshot present; Exercise 4 drift attacks documented with timings; Exercise 6 demonstrates a successful git-only change. |
| **Part 5 — Monitoring**          | 20     | Prometheus and Grafana running; custom dashboard with ≥3 panels submitted as JSON; one screenshot.                                                                                 |
| **Part 6 — Scenarios**           | 15     | All four scenarios answered. Marked on understanding of _why_ the tooling solves the problem, not on length.                                                                       |
| **Verification checklist**       | 5      | All 12 items ticked with matching output in `verification-output.txt`.                                                                                                             |

**Deductions:**

- Manual cluster edits in lieu of GitOps in Exercise 6: **−5**.
- Missing screenshots: **−2** each.
- Report exceeds 6 pages: **−3**.

---

## 9. Troubleshooting

### k3d cluster will not create

- Docker is not running. Start Docker Desktop (or `sudo service docker start` on native Linux) and wait for `docker info` to succeed.
- Port 8080 or 8443 is in use. Find the conflicting process with `sudo ss -tlnp | grep -E ':8080|:8443'` (or `sudo lsof -i :8080`), stop it, and re-run the `k3d cluster create` command. As a fallback, edit the cluster create command in §Part 1.A to use different host ports (e.g. `9080:8080` and `9543:8443`) and remember to use the new host port in every subsequent URL.
- On Windows + WSL: Docker Desktop's "WSL integration" is off. Open Docker Desktop → Settings → Resources → WSL Integration → enable for your Ubuntu distribution.

### k3d cluster created but has no port mappings, no agents, or Traefik is still installed

This means flags on `k3d cluster create` did not reach the binary. The most common cause is pasting a multi-line command into a shell that interprets the line-continuation character differently.

Confirm the problem:

```bash
k3d cluster list                                    # AGENTS column should be 2, not 0
docker ps --filter name=k3d-cisc814 --format '{{.Names}} {{.Ports}}'
# Expected: ports include 8080->8080 and 8443->8443. If those are missing, you hit this bug.
```

Fix — delete and re-create using the command in §Part 1.A exactly:

```bash
k3d cluster delete cisc814

k3d cluster create cisc814 \
  --servers 1 \
  --agents 2 \
  --port "8080:8080@loadbalancer" \
  --port "8443:8443@loadbalancer" \
  --k3s-arg "--disable=traefik@server:*" \
  --wait
```

### `kubectl` returns "the server doesn't have a resource type"

- Wrong context. `kubectl config current-context` should be `k3d-cisc814` (k3d) or `docker-desktop` (DDK8s). Switch with `kubectl config use-context <name>`.

### Pods stuck `ImagePullBackOff`

- Out of laptop disk. `docker system df` and `docker system prune -a` if needed.
- Corporate proxy blocking Docker Hub or ghcr.io. Configure Docker's proxy (Docker Desktop → Settings → Resources → Proxies; on native Linux edit `/etc/systemd/system/docker.service.d/http-proxy.conf`).

### Argo CD Application stuck `OutOfSync` after a push

- Argo CD's default refresh interval is 3 minutes. Click the Application → **Refresh** to force an immediate check.
- `repoURL` in the Application still points at `REPLACE-ME`. Re-run the rewrite from §3.7.
- Repository is private and Argo CD has no credentials. For this lab, make your fork public. In production you would add a `Repository` credential.

### Argo CD UI shows red exclamation on a child app

- Click the app → **Events** tab. The error is almost always (a) namespace does not exist (Argo CD does not create namespaces by default; the `CreateNamespace=true` sync option is set in our Application manifests, so this would mean an edited manifest), or (b) a typo in the `path` field.

### `kubectl top` says "metrics not available"

- metrics-server is not installed or not ready. Re-run §5.1. On Docker Desktop you must apply the `--kubelet-insecure-tls` patch.

### Grafana dashboards show "No data"

- Prometheus data source URL is wrong. In Grafana → Configuration → Data Sources → Prometheus, the URL should be `http://prometheus-server.monitoring.svc.cluster.local`. Our values file sets this automatically.
- Prometheus has not scraped a full interval yet. Wait 60 seconds.
- The dashboard expects a `cluster` label that Prometheus is not emitting. Edit panel → swap `cluster="$cluster"` for `instance=~".*"` as a quick fix.

### Self-healing did not revert my drift

- Check the Application's `syncPolicy.automated.selfHeal: true` is set (open the YAML in Argo CD UI). Without it, Argo CD detects drift but does not act.
- The Application is `OutOfSync` because of a _different_ drift you also introduced. Argo CD will revert all of it together on the next reconcile.

### Ingress URL returns "connection refused" or "404 Not Found"

- `kubectl get svc -n ingress-nginx ingress-nginx-controller` — the `EXTERNAL-IP` should be `localhost` and the port `8080`. If not, re-run the Helm install from §2.5.2.
- 404 from nginx means the Host header did not match any Ingress rule. `kubectl get ingress -A` and confirm your hostname is listed exactly as you typed it in the browser.
- On k3d, confirm the cluster was created with `--port "8080:8080@loadbalancer"` (not `8080:80`). If the mapping is wrong, delete the cluster and re-create it.

### Out of memory on an 8 GB laptop

- Stop other containers (`docker ps` → `docker stop ...`).
- Reduce Prometheus retention: edit [infrastructure/observability/prometheus/values.yaml](infrastructure/observability/prometheus/values.yaml) → `server.retention: 1d`, then:

  ```bash
  helm upgrade prometheus prometheus-community/prometheus \
    -n monitoring \
    -f infrastructure/observability/prometheus/values.yaml
  ```

- Re-create the cluster with 1 agent:

  ```bash
  k3d cluster delete cisc814
  k3d cluster create cisc814 \
    --servers 1 --agents 1 \
    --port "8080:8080@loadbalancer" \
    --port "8443:8443@loadbalancer" \
    --k3s-arg "--disable=traefik@server:*" \
    --wait
  ```

### How to start over completely

If you are on k3d, deleting the cluster removes everything in one shot:

```bash
k3d cluster delete cisc814
```

If you are on Docker Desktop Kubernetes, drop the lab's namespaces (the cluster itself stays running):

```bash
for ns in argocd ingress-nginx monitoring sample-web sample-web-dev sample-web-prod podinfo-dev podinfo-prod; do
  kubectl delete namespace "$ns" --ignore-not-found
done
```

---

## 10. Glossary

| Term                      | One-line definition                                                                    |
| ------------------------- | -------------------------------------------------------------------------------------- |
| **Cluster**               | A control plane plus a set of worker nodes that together run containerised workloads.  |
| **Node**                  | A single machine (VM, container, or physical) that runs Pods.                          |
| **Pod**                   | The smallest deployable unit — one or more containers that share network and storage.  |
| **Deployment**            | A controller that maintains a set of identical Pods and handles rollouts.              |
| **Service**               | A stable virtual IP + DNS name that load-balances across a label-selected set of Pods. |
| **Ingress**               | A routing rule that exposes a Service to the outside world via an Ingress controller.  |
| **Namespace**             | A scope for object names; used for tenancy, quotas, and RBAC.                          |
| **Label**                 | A key/value tag on an object.                                                          |
| **Selector**              | A query against labels that another object uses to find this one.                      |
| **ConfigMap**             | A key/value store mounted into Pods as env vars or files.                              |
| **Kustomize**             | Template-free YAML composition tool, built into `kubectl`.                             |
| **Helm**                  | Templated package manager for Kubernetes; uses "charts."                               |
| **Argo CD**               | Pull-based GitOps reconciler that runs inside the cluster.                             |
| **Application (Argo CD)** | A CR that says "render manifests from this git path and apply them to this namespace." |
| **Sync**                  | The act of making the cluster match git.                                               |
| **Drift**                 | Cluster state diverging from git state.                                                |
| **Self-healing**          | Argo CD automatically reverting drift back to git state.                               |
| **Prometheus**            | Pull-based metrics database and query engine.                                          |
| **Grafana**               | Dashboarding and visualisation layer for metrics backends.                             |
| **PromQL**                | Prometheus's query language.                                                           |
| **kube-state-metrics**    | An exporter that turns Kubernetes API objects into Prometheus metrics.                 |

---

_End of lab manual. Good luck._
