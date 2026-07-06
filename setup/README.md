# ArgoCD Local Setup Guide (Linux)

## Overview

This guide explains how to set up ArgoCD locally using a Kubernetes cluster (Kind), install the ArgoCD CLI, access the UI, and deploy your first application.

## Prerequisites

Install the following tools:

* Docker
* Kubernetes Cluster (Kind)
* kubectl
* Git
* ArgoCD CLI

Verify installations:

```bash
docker --version
kubectl version --client
git --version
argocd version
```

## Step 1: Start Kubernetes Cluster

### If using Kind

Create a cluster:

```bash
kind create cluster --name argocd-cluster
```

Verify:

```bash
kubectl get nodes
```

Expected:

```text
NAME                           STATUS
argocd-cluster-control-plane   Ready
```

## Step 2: Create ArgoCD Namespace

```bash
kubectl create namespace argocd
```

Verify:

```bash
kubectl get ns
```

## Step 3: Install ArgoCD

Install the official manifests:

```bash
kubectl apply -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Wait for all pods to become `Running`.

Check:

```bash
kubectl get pods -n argocd
```

## Step 4: Install ArgoCD CLI

Download the latest version:

```bash
VERSION=$(curl -L -s https://raw.githubusercontent.com/argoproj/argo-cd/stable/VERSION)

curl -sSL -o argocd \
https://github.com/argoproj/argo-cd/releases/download/$VERSION/argocd-linux-amd64
```

Make it executable:

```bash
chmod +x argocd
sudo mv argocd /usr/local/bin/
```

Verify:

```bash
argocd version
```

## Step 5: Access ArgoCD UI

Forward the ArgoCD server port:

```bash
kubectl port-forward svc/argocd-server \
-n argocd 8080:443
```

Keep this terminal running.

Open:

```text
https://localhost:8080
```

Ignore the browser security warning (self-signed certificate).

## Step 6: Get Initial Admin Password

Username:

```text
admin
```

Password:

```bash
kubectl -n argocd \
get secret argocd-initial-admin-secret \
-o jsonpath="{.data.password}" | base64 -d

echo
```

## Step 7: Login Using CLI

```bash
argocd login localhost:8080 --username admin --password PASTE_STEP_6 --insecure
```

Verify:

```bash
argocd account get-user-info
```

## Step 8: Verify Installation

```bash
kubectl get all -n argocd
```

You should see:

* argocd-server
* argocd-repo-server
* argocd-application-controller
* argocd-redis

## Repository Structure

Example:

```text
argoCD/

└── setup/
    ├── deployment.yaml
    └── service.yaml
```

## Create an Application

```bash
argocd app create nginx-demo \
  --repo https://github.com/<username>/<repo>.git \
  --path setup \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default
```

**Important:** `--dest-namespace` is required. Omitting it can lead to an `InvalidSpecError` indicating that the namespace is missing.

## Sync the Application

```bash
argocd app sync nginx-demo
```

## Check Application Status

```bash
argocd app get nginx-demo
```

## List Applications

```bash
argocd app list
```

## Refresh Application

```bash
argocd app refresh nginx-demo
```

## Enable Automatic Sync

```bash
argocd app set nginx-demo \
--sync-policy automated
```

This enables:

* Automatic deployment
* Self-healing
* GitOps reconciliation

## Delete an Application

```bash
argocd app delete nginx-demo
```

## Useful kubectl Commands

View nodes:

```bash
kubectl get nodes
```

View deployments:

```bash
kubectl get deploy
```

View services:

```bash
kubectl get svc
```

View pods:

```bash
kubectl get pods
```

View all resources:

```bash
kubectl get all
```

View ArgoCD resources:

```bash
kubectl get all -n argocd
```

Describe a pod:

```bash
kubectl describe pod <pod-name>
```

View logs:

```bash
kubectl logs <pod-name>
```

## Daily Workflow

### 1. Start Docker

```bash
sudo systemctl start docker
```

### 2. Start Kubernetes

Kind clusters usually start automatically when Docker starts. Verify:

```bash
kubectl get nodes
```

For Minikube:

```bash
minikube start
```

### 3. Start ArgoCD UI

```bash
kubectl port-forward svc/argocd-server \
-n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

### 4. Login (CLI)

```bash
argocd login localhost:8080 --insecure
```

### 5. Make Changes

Update your Kubernetes manifests.

Example:

```yaml
replicas: 3
```

Commit:

```bash
git add .
git commit -m "YOUR_MESSAGE"
git push
```

### 6. Sync

If Auto Sync is disabled:

```bash
argocd app sync nginx-demo
```

### 7. Verify

```bash
kubectl get pods
kubectl get deploy
```

## Common Errors

### InvalidSpecError

Example:

```text
Namespace for Deployment is missing
```

Fix:

```bash
argocd app set nginx-demo \
--dest-namespace default
```

or recreate the application with:

```bash
--dest-namespace default
```

### OutOfSync

The Git repository and Kubernetes cluster differ.

Fix:

```bash
argocd app sync nginx-demo
```

### Missing

The resource exists in Git but not in the cluster.

Usually resolved by:

```bash
argocd app sync nginx-demo
```

### Repository Not Accessible

Possible causes:

* Wrong Git URL
* Private repository
* Authentication issues

Verify:

```bash
argocd app get nginx-demo
```

### Pods Stuck in ContainerCreating

Inspect:

```bash
kubectl describe pod <pod-name>
```

### Cannot Access UI

Restart port forwarding:

```bash
kubectl port-forward svc/argocd-server \
-n argocd 8080:443
```

## Important Commands Cheat Sheet

```bash
kubectl get nodes

kubectl get pods

kubectl get all -n argocd

argocd login localhost:8080 --insecure

argocd app list

argocd app get nginx-demo

argocd app sync nginx-demo

argocd app refresh nginx-demo

argocd app delete nginx-demo
```

## GitOps Workflow

```text
Developer
     │
     │ git push
     ▼
Git Repository
     │
     ▼
ArgoCD Repository Server
     │
     ▼
Application Controller
     │
     ▼
Kubernetes API Server
     │
     ▼
Pods / Services / Deployments
```

ArgoCD continuously compares:

* Desired state (Git)
* Actual state (Kubernetes)

If they differ, ArgoCD marks the application as **OutOfSync**. With automated sync enabled, it reconciles the cluster back to the desired state automatically.
