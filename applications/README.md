# Argo CD Applications

This folder contains Argo CD `Application` manifests for the main GitOps demo.

## Applications

- `frontend-app.yaml` deploys `apps/frontend`
- `backend-app.yaml` deploys `apps/backend`
- `database-app.yaml` deploys `apps/database`

Each app deploys into the `gitops-demo` namespace and uses automated sync with pruning and self-healing.

## Folder Layout

```text
applications/
├── frontend-app.yaml
├── backend-app.yaml
└── database-app.yaml

apps/
├── frontend/
├── backend/
└── database/
```

Each folder under `apps/` has:

- `deployment.yaml`
- `service.yaml`

## Apply The Applications

From the repository root:

```bash
kubectl apply -f applications/frontend-app.yaml
kubectl apply -f applications/backend-app.yaml
kubectl apply -f applications/database-app.yaml
```

Or apply the whole folder:

```bash
kubectl apply -f applications/
```

## Check Status

```bash
argocd app list
argocd app get frontend
argocd app get backend
argocd app get database
kubectl get all -n gitops-demo
```

## Repo URL

The manifests currently use:

```yaml
repoURL: https://github.com/sp-tnex/argoCD.git
```

If your repository URL is different, update `repoURL` in all three Application manifests, then commit and push before syncing in Argo CD.

See the root `README.md` for the full Kind setup guide, local run guide, cleanup steps, and Argo CD learning notes.
