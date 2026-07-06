# Argo CD Notes

These notes are for remembering Argo CD quickly in the future. Keep this file as a study and command reference separate from the project `README.md`.

## One-Line Answer

Argo CD is a GitOps continuous delivery tool for Kubernetes. It watches a Git repository, compares the desired state in Git with the live state in the cluster, and syncs the cluster so it matches Git.

## What Is Argo CD?

Argo CD is a declarative deployment tool for Kubernetes.

In normal Kubernetes work, we run commands like:

```bash
kubectl apply -f deployment.yaml
```

That works, but it depends on a human or CI job applying the correct files at the correct time.

With Argo CD, Git becomes the source of truth. We store Kubernetes manifests, Helm charts, or Kustomize overlays in Git. Argo CD continuously checks Git and the Kubernetes cluster. If the cluster does not match Git, Argo CD shows the difference and can sync it.

## Why Argo CD Is Used

Argo CD is used because teams want predictable, auditable, repeatable Kubernetes deployments.

Important reasons:

- Git becomes the single source of truth.
- Every deployment change has a Git commit history.
- Rollback is easier because we can revert Git commits.
- Argo CD detects drift when someone changes the cluster manually.
- Argo CD can self-heal drift automatically.
- Argo CD UI shows application health, sync status, resources, and errors.
- Teams can manage many apps and clusters from one place.
- It supports Kubernetes YAML, Helm, Kustomize, Jsonnet, and plugins.
- It fits GitOps best practices.

## What Is GitOps?

GitOps means using Git as the source of truth for infrastructure and application delivery.

Basic GitOps flow:

```text
Developer changes YAML, Helm, or Kustomize files
        |
        v
Change is committed and pushed to Git
        |
        v
Argo CD detects the Git change
        |
        v
Argo CD compares Git desired state with cluster live state
        |
        v
Argo CD syncs Kubernetes resources
```

GitOps has two common models:

- Push model: CI pushes changes directly to the cluster.
- Pull model: a tool inside the cluster pulls desired state from Git.

Argo CD follows the pull model.

## Argo CD Architecture

Main components:

- `argocd-server`: API server, web UI, and CLI entry point.
- `argocd-application-controller`: watches Argo CD Applications and reconciles desired state with live cluster state.
- `argocd-repo-server`: clones Git repos and renders manifests from YAML, Helm, Kustomize, or plugins.
- `argocd-redis`: cache used by Argo CD.
- `argocd-dex-server`: optional identity provider integration for SSO.
- `argocd-notifications-controller`: optional component for notifications.
- `argocd-applicationset-controller`: optional controller for generating many Applications from templates.

Architecture flow:

```text
Git repository
     |
     v
argocd-repo-server renders manifests
     |
     v
argocd-application-controller compares desired and live state
     |
     v
Kubernetes API server applies changes
     |
     v
Pods, Services, Deployments, ConfigMaps, Secrets
```

## Key Argo CD Concepts

`Application`: Main Argo CD custom resource. It defines what to deploy, from where, and to which cluster/namespace.

`AppProject`: A security and organization boundary. It controls allowed repos, clusters, namespaces, and resource kinds.

`Source`: Git repo, path, revision, Helm chart, or Kustomize config.

`Destination`: Target Kubernetes cluster and namespace.

`Sync`: Applying the Git desired state into the cluster.

`Refresh`: Rechecking Git and cluster state.

`Health`: Whether resources are working correctly.

`Sync Status`: Whether live cluster state matches Git.

`OutOfSync`: Live cluster differs from Git.

`Synced`: Live cluster matches Git.

`Healthy`: Resources are running correctly.

`Degraded`: Resources have a problem.

`Missing`: Resource exists in Git but not in the cluster.

`Prune`: Delete resources from the cluster when they are removed from Git.

`Self-Heal`: Automatically fix drift caused by manual cluster changes.

`Rollback`: Return to a previous deployed version.

`Diff`: Show what is different between Git and the live cluster.

## Application Manifest Example

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: frontend
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/username/repo.git
    targetRevision: HEAD
    path: apps/frontend
  destination:
    server: https://kubernetes.default.svc
    namespace: gitops-demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

Important fields:

- `metadata.name`: Argo CD application name.
- `metadata.namespace`: usually `argocd`, where Argo CD is installed.
- `spec.project`: Argo CD project name.
- `spec.source.repoURL`: Git repository URL.
- `spec.source.targetRevision`: branch, tag, commit SHA, or `HEAD`.
- `spec.source.path`: folder path inside the repo.
- `spec.destination.server`: target Kubernetes cluster API.
- `spec.destination.namespace`: target namespace.
- `spec.syncPolicy.automated`: enables automatic sync.
- `prune: true`: deletes resources removed from Git.
- `selfHeal: true`: fixes manual drift.
- `CreateNamespace=true`: creates destination namespace if missing.

## Sync Policies

Manual sync:

- Argo CD detects changes but waits for a user to sync.
- Useful for learning and controlled environments.

Automated sync:

- Argo CD syncs Git changes automatically.
- Useful for production GitOps when changes are reviewed before merge.

Automated sync with prune:

- Removes cluster resources that were deleted from Git.

Automated sync with self-heal:

- Fixes manual changes made directly in the cluster.

Example:

```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
```

## Common Sync Options

```yaml
syncOptions:
  - CreateNamespace=true
  - PruneLast=true
  - ApplyOutOfSyncOnly=true
  - ServerSideApply=true
```

Meanings:

- `CreateNamespace=true`: create destination namespace automatically.
- `PruneLast=true`: delete removed resources after applying other changes.
- `ApplyOutOfSyncOnly=true`: only apply resources that are out of sync.
- `ServerSideApply=true`: use Kubernetes server-side apply.

## Installation Commands

Create Kind cluster:

```bash
kind create cluster --name argocd-cluster
kubectl config use-context kind-argocd-cluster
kubectl get nodes
```

Install Argo CD:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=300s
```

Check Argo CD:

```bash
kubectl get pods -n argocd
kubectl get svc -n argocd
kubectl get all -n argocd
```

## Login Commands

Port-forward Argo CD UI:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open:

```text
https://localhost:8080
```

Get admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
echo
```

Login:

```bash
argocd login localhost:8080 --username admin --password <password> --insecure
```

Check account:

```bash
argocd account get-user-info
```

Change admin password:

```bash
argocd account update-password
```

## Application Commands

Create an app using CLI:

```bash
argocd app create frontend \
  --repo https://github.com/username/repo.git \
  --path apps/frontend \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace gitops-demo
```

List apps:

```bash
argocd app list
```

Get app details:

```bash
argocd app get frontend
```

Sync app:

```bash
argocd app sync frontend
```

Refresh app:

```bash
argocd app refresh frontend
```

Hard refresh app:

```bash
argocd app get frontend --hard-refresh
```

Show diff:

```bash
argocd app diff frontend
```

Show history:

```bash
argocd app history frontend
```

Rollback:

```bash
argocd app rollback frontend <history-id>
```

Delete app:

```bash
argocd app delete frontend
```

Set automated sync:

```bash
argocd app set frontend --sync-policy automated
```

Enable auto-prune:

```bash
argocd app set frontend --auto-prune
```

Enable self-heal:

```bash
argocd app set frontend --self-heal
```

Disable automated sync:

```bash
argocd app set frontend --sync-policy none
```

Wait for app:

```bash
argocd app wait frontend --health --sync --timeout 300
```

## Kubernetes Commands For Argo CD Apps

Apply Application manifests:

```bash
kubectl apply -f applications/
```

List Argo CD Applications:

```bash
kubectl get applications -n argocd
```

Describe an Application:

```bash
kubectl describe application frontend -n argocd
```

Get Application YAML:

```bash
kubectl get application frontend -n argocd -o yaml
```

Check app namespace:

```bash
kubectl get all -n gitops-demo
```

Check events:

```bash
kubectl get events -n gitops-demo --sort-by=.lastTimestamp
kubectl get events -n argocd --sort-by=.lastTimestamp
```

Check pods:

```bash
kubectl get pods -n gitops-demo
kubectl describe pod <pod-name> -n gitops-demo
kubectl logs <pod-name> -n gitops-demo
```

## Repository Commands

Add a Git repo to Argo CD:

```bash
argocd repo add https://github.com/username/repo.git
```

For a private HTTPS repo:

```bash
argocd repo add https://github.com/username/private-repo.git \
  --username <username> \
  --password <token>
```

For SSH:

```bash
argocd repo add git@github.com:username/private-repo.git \
  --ssh-private-key-path ~/.ssh/id_rsa
```

List repos:

```bash
argocd repo list
```

Check repo connection:

```bash
argocd repo get https://github.com/username/repo.git
```

Remove repo:

```bash
argocd repo rm https://github.com/username/repo.git
```

## Cluster Commands

List clusters known to Argo CD:

```bash
argocd cluster list
```

Add an external cluster:

```bash
argocd cluster add <kubectl-context-name>
```

Remove a cluster:

```bash
argocd cluster rm <cluster-name-or-server>
```

For local in-cluster deployment, this destination is common:

```yaml
destination:
  server: https://kubernetes.default.svc
```

## Project Commands

List projects:

```bash
argocd proj list
```

Create project:

```bash
argocd proj create demo
```

Get project:

```bash
argocd proj get demo
```

Allow repo in project:

```bash
argocd proj add-source demo https://github.com/username/repo.git
```

Allow destination:

```bash
argocd proj add-destination demo https://kubernetes.default.svc gitops-demo
```

Delete project:

```bash
argocd proj delete demo
```

## Users And RBAC Commands

List accounts:

```bash
argocd account list
```

Get current user info:

```bash
argocd account get-user-info
```

RBAC is usually configured in the `argocd-rbac-cm` ConfigMap:

```bash
kubectl get configmap argocd-rbac-cm -n argocd -o yaml
```

Example RBAC idea:

```yaml
policy.csv: |
  p, role:developer, applications, get, */*, allow
  p, role:developer, applications, sync, */*, allow
  g, alice, role:developer
```

## Health And Sync Status

Common sync statuses:

- `Synced`: live state matches Git.
- `OutOfSync`: live state differs from Git.
- `Unknown`: Argo CD cannot determine sync state.

Common health statuses:

- `Healthy`: app resources look good.
- `Progressing`: rollout is still happening.
- `Degraded`: resource is failing.
- `Suspended`: resource is paused.
- `Missing`: expected resource is missing.
- `Unknown`: health cannot be determined.

Useful command:

```bash
argocd app get frontend
```

## Drift Detection And Self-Healing

Drift means live cluster state is different from Git.

Example drift:

```bash
kubectl scale deployment frontend -n gitops-demo --replicas=5
```

If Git says replicas should be `2`, Argo CD will show `OutOfSync`.

If `selfHeal: true` is enabled, Argo CD changes it back to `2`.

## Pruning

Pruning means deleting resources from the cluster when they are removed from Git.

Without prune:

- Resource is deleted from Git.
- Resource may still remain in the cluster.

With prune:

- Resource is deleted from Git.
- Argo CD deletes it from the cluster during sync.

Use carefully in production.

## Rollback

Argo CD rollback returns an app to a previous deployment history entry.

Commands:

```bash
argocd app history frontend
argocd app rollback frontend <history-id>
```

GitOps-friendly rollback is usually:

```bash
git revert <bad-commit-sha>
git push
```

Then Argo CD syncs the reverted desired state.

## Helm With Argo CD

Argo CD can deploy Helm charts.

Example Application source:

```yaml
source:
  repoURL: https://charts.bitnami.com/bitnami
  chart: nginx
  targetRevision: 15.0.0
  helm:
    values: |
      replicaCount: 2
```

Common Helm commands outside Argo CD:

```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm template nginx bitnami/nginx
```

In GitOps, prefer storing Helm values in Git.

## Kustomize With Argo CD

Argo CD can deploy Kustomize overlays.

Example structure:

```text
apps/frontend/base/
apps/frontend/overlays/dev/
apps/frontend/overlays/prod/
```

Example Application path:

```yaml
source:
  repoURL: https://github.com/username/repo.git
  targetRevision: HEAD
  path: apps/frontend/overlays/dev
```

Useful Kustomize check:

```bash
kubectl kustomize apps/frontend/overlays/dev
```

## ApplicationSet

ApplicationSet generates multiple Argo CD Applications from a template.

Use it when:

- You have many apps with the same structure.
- You deploy one app to many clusters.
- You deploy one app to many environments.
- You want applications generated from Git directories.

Common generators:

- List generator
- Git generator
- Cluster generator
- Matrix generator
- Pull request generator

Simple idea:

```text
One ApplicationSet
        |
        v
Many generated Applications
```

## Multi-Environment Pattern

Common repo layout:

```text
apps/
  frontend/
    base/
    overlays/
      dev/
      staging/
      prod/
```

Dev app points to:

```yaml
path: apps/frontend/overlays/dev
```

Prod app points to:

```yaml
path: apps/frontend/overlays/prod
```

## App Of Apps Pattern

The app-of-apps pattern means one parent Argo CD Application deploys child Argo CD Applications.

Example:

```text
root-app
  deploys applications/frontend-app.yaml
  deploys applications/backend-app.yaml
  deploys applications/database-app.yaml
```

This is useful when many apps should be bootstrapped together.

## Secrets In Argo CD

Do not store raw production secrets in Git.

Better options:

- Sealed Secrets
- External Secrets Operator
- SOPS with age or GPG
- Vault integration
- Cloud secret managers

Bad:

```yaml
POSTGRES_PASSWORD: plain-text-password
```

Better for real projects:

```yaml
valueFrom:
  secretKeyRef:
    name: database-secret
    key: password
```

This demo uses simple values only for local learning.

## Security Notes

Important security practices:

- Use SSO instead of sharing admin credentials.
- Disable or restrict the admin account in production.
- Use AppProjects to restrict repos and destinations.
- Use RBAC to control who can view, sync, delete, or manage apps.
- Avoid storing plain secrets in Git.
- Review pull requests before merging deployment changes.
- Use separate projects for teams or environments.
- Be careful with auto-prune in shared clusters.

## Troubleshooting

App is `OutOfSync`:

```bash
argocd app diff frontend
argocd app get frontend
argocd app sync frontend
```

App is `Degraded`:

```bash
kubectl get pods -n gitops-demo
kubectl describe pod <pod-name> -n gitops-demo
kubectl logs <pod-name> -n gitops-demo
```

Repo connection problem:

```bash
argocd repo list
argocd repo get <repo-url>
kubectl logs deployment/argocd-repo-server -n argocd
```

Application controller problem:

```bash
kubectl logs statefulset/argocd-application-controller -n argocd
```

Argo CD server problem:

```bash
kubectl logs deployment/argocd-server -n argocd
```

Namespace missing:

```bash
kubectl get ns
kubectl create namespace gitops-demo
```

Or add:

```yaml
syncOptions:
  - CreateNamespace=true
```

CRD not found:

```bash
kubectl get crds
kubectl apply -f <crd-manifest.yaml>
```

Image pull issue:

```bash
kubectl describe pod <pod-name> -n gitops-demo
kubectl get events -n gitops-demo --sort-by=.lastTimestamp
```

## Common Interview Answers

Question: What problem does Argo CD solve?

Answer: It automates Kubernetes deployments using Git as the source of truth. It detects drift, shows differences, and syncs the cluster to the desired state stored in Git.

Question: How is Argo CD different from Jenkins?

Answer: Jenkins is commonly used for CI and can push deployments. Argo CD is a Kubernetes GitOps CD tool that pulls desired state from Git and continuously reconciles the cluster.

Question: What is sync in Argo CD?

Answer: Sync means applying the desired state from Git into the Kubernetes cluster.

Question: What is self-heal?

Answer: Self-heal means Argo CD automatically fixes manual changes in the cluster so the live state matches Git again.

Question: What is prune?

Answer: Prune means Argo CD deletes live cluster resources that were removed from Git.

Question: What is an Application?

Answer: An Application is an Argo CD custom resource that defines the Git source, target revision, path, destination cluster, namespace, and sync policy.

Question: What is an AppProject?

Answer: An AppProject is a boundary for grouping applications and controlling which repos, clusters, namespaces, and Kubernetes resource kinds are allowed.

Question: What is drift?

Answer: Drift is when the live Kubernetes cluster differs from the desired state stored in Git.

Question: What is the app-of-apps pattern?

Answer: It is a pattern where one parent Argo CD Application deploys multiple child Argo CD Applications.

Question: What is ApplicationSet?

Answer: ApplicationSet is a controller that generates many Argo CD Applications from templates and generators.

## Daily Command Cheat Sheet

```bash
# Cluster and Argo CD
kubectl get pods -n argocd
kubectl get svc -n argocd
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Login
argocd login localhost:8080 --username admin --password <password> --insecure
argocd account get-user-info

# Apps
argocd app list
argocd app get frontend
argocd app diff frontend
argocd app sync frontend
argocd app refresh frontend
argocd app history frontend
argocd app rollback frontend <history-id>
argocd app delete frontend

# Kubernetes resources
kubectl get applications -n argocd
kubectl describe application frontend -n argocd
kubectl get all -n gitops-demo
kubectl get events -n gitops-demo --sort-by=.lastTimestamp
kubectl logs <pod-name> -n gitops-demo
kubectl describe pod <pod-name> -n gitops-demo

# Repos and clusters
argocd repo list
argocd repo add <repo-url>
argocd cluster list
argocd cluster add <context-name>

# Projects
argocd proj list
argocd proj create demo
argocd proj get demo
```

## Best Practices To Remember

- Keep all Kubernetes desired state in Git.
- Use pull requests for every deployment change.
- Keep app manifests small and organized.
- Use separate environments for dev, staging, and production.
- Use AppProjects for security boundaries.
- Use automated sync only after the Git review process is solid.
- Use prune carefully.
- Do not store real secrets as plain text.
- Prefer Git revert for rollback.
- Monitor Argo CD app health and sync status.
- Avoid making manual changes directly in the cluster.

## Useful Links

- Argo CD docs: <https://argo-cd.readthedocs.io/>
- Argo CD getting started: <https://argo-cd.readthedocs.io/en/stable/getting_started/>
- Argo CD Application spec: <https://argo-cd.readthedocs.io/en/stable/user-guide/application-specification/>
- Argo CD ApplicationSet: <https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/>
- Kubernetes docs: <https://kubernetes.io/docs/home/>
- Kind docs: <https://kind.sigs.k8s.io/docs/user/quick-start/>
