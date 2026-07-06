# Argo CD GitOps Project

This repository is a small GitOps project for learning Argo CD with a local Kind Kubernetes cluster.

It deploys three apps:

- `frontend`: NGINX web server
- `backend`: simple HTTP API response service
- `database`: PostgreSQL database

Argo CD watches this Git repository and keeps the Kubernetes cluster matching the manifests in Git.

## Repository Structure

```text
argocd-project/
├── apps/
│   ├── frontend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   ├── backend/
│   │   ├── deployment.yaml
│   │   └── service.yaml
│   └── database/
│       ├── deployment.yaml
│       └── service.yaml
├── applications/
│   ├── frontend-app.yaml
│   ├── backend-app.yaml
│   └── database-app.yaml
└── README.md
```

The `apps/` directory contains Kubernetes manifests.

The `applications/` directory contains Argo CD `Application` manifests. Each Argo CD application points to one folder under `apps/`.

## Prerequisites

Install these tools:

- Docker
- Kind
- kubectl
- Git
- Argo CD CLI

Check them:

```bash
docker --version
kind --version
kubectl version --client
git --version
argocd version --client
```

## Important: Update The Repo URL

The Argo CD Application manifests currently point to:

```yaml
repoURL: https://github.com/sp-tnex/argoCD.git
```

If your GitHub repository URL is different, update `repoURL` in:

- `applications/frontend-app.yaml`
- `applications/backend-app.yaml`
- `applications/database-app.yaml`

Argo CD reads from Git, so commit and push changes before syncing from Argo CD.

## Local Setup With Kind

Create a local Kubernetes cluster:

```bash
kind create cluster --name argocd-cluster
kubectl config use-context kind-argocd-cluster
kubectl get nodes
```

Create the Argo CD namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait until Argo CD is ready:

```bash
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
kubectl get pods -n argocd
```

## Open Argo CD UI

Start port-forwarding:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open this URL:

```text
https://localhost:8080
```

The browser may show a certificate warning. That is expected for this local setup.

Get the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo
```

Login:

```bash
argocd login localhost:8080 --username admin --password <password> --insecure
```

## Deploy The Project With Argo CD

From the repository root, apply the Argo CD Application manifests:

```bash
kubectl apply -f applications/frontend-app.yaml
kubectl apply -f applications/backend-app.yaml
kubectl apply -f applications/database-app.yaml
```

Or apply the whole folder:

```bash
kubectl apply -f applications/
```

Argo CD will create the `gitops-demo` namespace because each Application has:

```yaml
syncOptions:
  - CreateNamespace=true
```

Check Argo CD apps:

```bash
argocd app list
argocd app get frontend
argocd app get backend
argocd app get database
```

If automatic sync has not started yet, sync manually:

```bash
argocd app sync frontend
argocd app sync backend
argocd app sync database
```

Check Kubernetes resources:

```bash
kubectl get all -n gitops-demo
```

## Run And Test Locally

Port-forward the frontend:

```bash
kubectl port-forward svc/frontend -n gitops-demo 8081:80
```

Open:

```text
http://localhost:8081
```

Port-forward the backend:

```bash
kubectl port-forward svc/backend -n gitops-demo 8082:8080
```

Test it:

```bash
curl http://localhost:8082
```

Port-forward the database:

```bash
kubectl port-forward svc/database -n gitops-demo 5432:5432
```

Connect with `psql` if it is installed:

```bash
PGPASSWORD=apppassword psql -h localhost -p 5432 -U appuser -d appdb
```

## GitOps Workflow

Use this flow when changing the project:

1. Edit manifests in `apps/`.
2. Commit and push to Git.
3. Argo CD detects the change.
4. Argo CD syncs the cluster to match Git.
5. Verify with `argocd app get <app-name>` and `kubectl get all -n gitops-demo`.

Example change:

```bash
kubectl scale deployment frontend -n gitops-demo --replicas=5
```

Because `selfHeal: true` is enabled, Argo CD should move the cluster back to the replica count declared in Git.

## What Is Argo CD?

Argo CD is a declarative continuous delivery tool for Kubernetes. It uses Git repositories as the source of truth for Kubernetes manifests, Helm charts, Kustomize overlays, or other supported configuration formats.

Instead of manually applying manifests again and again, you describe the desired state in Git. Argo CD compares that desired state with the live cluster and shows whether the app is `Synced`, `OutOfSync`, `Healthy`, or `Degraded`.

## Why Use Argo CD?

Argo CD is useful because it gives you:

- Git as the source of truth
- Clear deployment history through Git commits
- Automatic or manual sync
- Drift detection when the cluster differs from Git
- Self-healing for unwanted manual changes
- Rollback by reverting Git commits
- A web UI for visualizing application health and sync state
- Multi-application and multi-cluster deployment patterns

## Argo CD Architecture

Main components:

- `argocd-server`: API server and web UI
- `argocd-application-controller`: watches Applications and reconciles cluster state
- `argocd-repo-server`: fetches Git repositories and renders manifests
- `argocd-redis`: cache used by Argo CD components
- `argocd-dex-server`: optional identity provider integration

Basic flow:

```text
Developer pushes to Git
        |
        v
Argo CD repo-server reads Git
        |
        v
Application controller compares Git state with cluster state
        |
        v
Argo CD syncs Kubernetes resources
        |
        v
Cluster matches Git
```

## Key Argo CD Concepts

`Application`: An Argo CD custom resource that tells Argo CD what repo path to deploy and where to deploy it.

`Project`: A grouping and policy boundary for Applications.

`Sync`: The action that applies desired Git state into the cluster.

`Health`: Argo CD's view of whether Kubernetes resources are running correctly.

`Drift`: A difference between the live cluster and the desired state in Git.

`Self-heal`: Automatic correction when live cluster resources drift from Git.

`Prune`: Automatic removal of resources that were deleted from Git.

## Useful Commands

```bash
argocd app list
argocd app get frontend
argocd app diff frontend
argocd app sync frontend
argocd app history frontend
argocd app rollback frontend <history-id>
kubectl get applications -n argocd
kubectl describe application frontend -n argocd
kubectl get all -n gitops-demo
```

## Cleanup

Delete the Argo CD Applications:

```bash
kubectl delete -f applications/frontend-app.yaml
kubectl delete -f applications/backend-app.yaml
kubectl delete -f applications/database-app.yaml
```

Delete the demo namespace:

```bash
kubectl delete namespace gitops-demo
```

Delete the Kind cluster:

```bash
kind delete cluster --name argocd-cluster
```
