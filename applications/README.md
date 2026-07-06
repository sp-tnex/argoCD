# Multiple Argo CD Applications

This folder contains three Argo CD `Application` manifests:

- `nginx-app` deploys `practice/nginx`
- `redis-app` deploys `practice/redis`
- `httpd-app` deploys `practice/httpd`

Each app is separate on purpose. You can sync, refresh, troubleshoot, and delete one app without touching the others.

## Folder Layout

```text
applications/
├── nginx-app.yaml
├── redis-app.yaml
└── httpd-app.yaml

practice/
├── nginx/
├── redis/
└── httpd/
```

Each folder under `practice/` has its own Kubernetes manifests:

- `namespace.yaml`
- `deployment.yaml`
- `service.yaml`

## Fresh Local Setup

Use this section when starting from zero on your local machine.

You need these tools installed first:

- Docker
- Kind
- kubectl
- Argo CD CLI
- Git

Check them:

```bash
docker --version
kind --version
kubectl version --client
argocd version --client
git --version
```

### 1. Create A Local Kind Cluster

Create the cluster:

```bash
kind create cluster --name argocd-cluster
```

Kind will set your kubectl context to:

```text
kind-argocd-cluster
```

Check the cluster:

```bash
kubectl cluster-info --context kind-argocd-cluster
kubectl get nodes
```

Do not run this:

```bash
kubectl cluster-info --context
```

`--context` needs a value after it. If you do not want to pass the context every time, use:

```bash
kubectl config use-context kind-argocd-cluster
```

### 2. Install Argo CD

After creating the Kind cluster, Argo CD is not installed yet. That is why this command can fail:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

If you see this error:

```text
Error from server (NotFound): namespaces "argocd" not found
```

It means the cluster exists, but the `argocd` namespace has not been created yet.

Create the namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for Argo CD pods:

```bash
kubectl get pods -n argocd
```

Most pods may show `ContainerCreating` for a little while. Wait until they are running:

```bash
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

Check the Argo CD services:

```bash
kubectl get svc -n argocd
```

You should see `argocd-server`.

### 3. Open Argo CD Locally

Run this in a separate terminal and keep it running:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

The browser may show a certificate warning. That is expected for this local setup.

### 4. Login To Argo CD

Username:

```text
admin
```

Get the initial password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo
```

Login with the CLI:

```bash
argocd login localhost:8080 --username admin --password <password> --insecure
```

Check login:

```bash
argocd account get-user-info
```

## Quick Setup Checklist

If you already know the steps, this is the short version:

```bash
kind create cluster --name argocd-cluster
kubectl config use-context kind-argocd-cluster

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s

kubectl port-forward svc/argocd-server -n argocd 8080:443
```

In another terminal:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo

argocd login localhost:8080 --username admin --password <password> --insecure
```

## Important: Repo URL

The Application manifests currently use this repo:

```yaml
repoURL: https://github.com/sp-tnex/argoCD.git
```

If you fork or rename the repository, update `repoURL` in all three files:

- `applications/nginx-app.yaml`
- `applications/redis-app.yaml`
- `applications/httpd-app.yaml`

Argo CD reads from Git, so push your changes before syncing from Argo CD.

## Create The Applications

From the repo root, run:

```bash
kubectl apply -f applications/
```

Check that Argo CD can see them:

```bash
argocd app list
```

You should see:

```text
nginx-app
redis-app
httpd-app
```

## Sync Each App Separately

Sync Nginx:

```bash
argocd app sync nginx-app
```

Sync Redis:

```bash
argocd app sync redis-app
```

Sync Httpd:

```bash
argocd app sync httpd-app
```

This is the main learning point: each app has its own sync. A change in `practice/redis` only needs `redis-app` to be synced.

## Check Status

Check all apps:

```bash
argocd app list
```

Check one app in detail:

```bash
argocd app get nginx-app
argocd app get redis-app
argocd app get httpd-app
```

Check Kubernetes resources:

```bash
kubectl get all -n nginx
kubectl get all -n redis
kubectl get all -n httpd
```

## Refresh After Git Changes

If you pushed a new commit and Argo CD has not noticed it yet:

```bash
argocd app refresh nginx-app
argocd app refresh redis-app
argocd app refresh httpd-app
```

Then sync the app you changed:

```bash
argocd app sync redis-app
```

## Example Learning Flow

Try this when practicing:

1. Change the Redis image or replica count in `practice/redis/deployment.yaml`.
2. Commit and push the change.
3. Refresh only `redis-app`.
4. Sync only `redis-app`.
5. Check that `nginx-app` and `httpd-app` were not changed.

Useful commands:

```bash
git status
git add practice/redis/deployment.yaml
git commit -m "Update redis deployment"
git push

argocd app refresh redis-app
argocd app sync redis-app
argocd app get redis-app
```

## Delete Only The Apps

Delete only one app:

```bash
argocd app delete redis-app
```

Delete all three:

```bash
argocd app delete nginx-app
argocd app delete redis-app
argocd app delete httpd-app
```

If you also want to remove the namespaces:

```bash
kubectl delete namespace nginx redis httpd
```

## What This Practice Teaches

- One Git repo can hold many applications.
- Each Argo CD `Application` points to one path.
- Each app can be synced independently.
- Smaller apps are easier to debug than one large shared app.
- Manual sync is useful for learning because you control when each app changes.

## Complete Local Cleanup

Use this when you are done practicing and want to stop everything running locally.

### 1. Stop Port Forwarding

If this command is still running in a terminal:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Stop it with:

```text
Ctrl + C
```

### 2. Delete The Argo CD Applications

This removes the Argo CD app records and the workloads they manage:

```bash
argocd app delete nginx-app
argocd app delete redis-app
argocd app delete httpd-app
```

Check:

```bash
argocd app list
```

### 3. Delete Practice Namespaces

If any practice namespaces are still present, delete them:

```bash
kubectl delete namespace nginx redis httpd
```

Check:

```bash
kubectl get ns
```

### 4. Delete Argo CD

If you want to remove Argo CD from the cluster:

```bash
kubectl delete namespace argocd
```

Check:

```bash
kubectl get ns
```

### 5. Delete The Kind Cluster

This is the cleanest final step. It removes the local Kubernetes cluster completely:

```bash
kind delete cluster --name argocd-cluster
```

Check that the cluster is gone:

```bash
kind get clusters
kubectl config get-contexts
```

If `argocd-cluster` is gone from `kind get clusters`, the local practice cluster is no longer running.

### Quick Cleanup

If you just want to remove everything from this practice quickly:

```bash
argocd app delete nginx-app
argocd app delete redis-app
argocd app delete httpd-app

kubectl delete namespace nginx redis httpd argocd
kind delete cluster --name argocd-cluster
```

If the Kind cluster is deleted, you do not need to separately delete namespaces. Deleting the cluster removes everything inside it. The separate namespace commands are useful when you want to clean resources but keep the cluster.
