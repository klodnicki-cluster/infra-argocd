# ArgoCD Bootstrap Guide

This guide explains how to initially set up ArgoCD on your Kubernetes cluster using this repository. After the initial bootstrap, ArgoCD will automatically discover and manage all repositories tagged with the `argocd` GitHub topic.

## Prerequisites

- Kubernetes cluster (1.24+)
- `kubectl` configured to access your cluster
- `helm` CLI (3.10+)
- Git access to this repository
- GitHub organization with repositories to deploy
- GitHub token with repository read access

## Initial Bootstrap Steps

### Step 1: Add the ArgoCD Helm Repository

```bash
helm repo add argo-cd https://argoproj.github.io/argo-helm
helm repo update
```

### Step 2: Create the ArgoCD Namespace

```bash
kubectl create namespace argocd
```

### Step 3: Install ArgoCD Helm Chart

First, fetch the chart dependencies:

```bash
cd argocd/
helm dependency update
```

Then install ArgoCD:

```bash
helm install argocd argo-cd/ \
  --namespace argocd \
  --values values.yaml
```

Or, using the Helm chart directly from the repo:

```bash
helm install argocd argo-cd/argo-cd \
  --namespace argocd \
  --values argocd/values.yaml \
  --version 7.3.3
```

### Step 5: Create GitHub Token Secret

The ApplicationSet needs a GitHub token to discover repositories with the `argocd` topic.

#### Create a GitHub Personal Access Token (PAT)

1. Go to GitHub Settings → Developer settings → Personal access tokens
2. Generate a new token (classic)
3. Grant `public_repo` scope (or `repo` for private repos)
4. Copy the token

#### Create the secret in Kubernetes

```bash
kubectl create secret generic github-token \
  --from-literal=token='<your-github-token>' \
  -n argocd
```

### Step 6: Verify the ApplicationSet

The ApplicationSet is part of the Helm chart and was automatically deployed during installation.

```bash
kubectl get applicationsets -n argocd
kubectl describe applicationset argocd-deployments -n argocd
```

### Step 7: Wait for ApplicationSet to Discover Repositories

The ApplicationSet will:
1. Poll GitHub for repositories in your organization with the `argocd` topic
2. Generate Applications for each discovered repository
3. ArgoCD will deploy them

This typically takes a few minutes. Watch the discovery:

```bash
# Watch for generated Applications
kubectl get applications -n argocd -w
```

## Accessing ArgoCD

### Port-Forward (Development/Testing)

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then access: https://localhost:8080

### Get Initial Admin Password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

Username: `admin`

### Configure for Production Access

For production, configure:
1. **Ingress**: Create an Ingress resource pointing to `argocd-server`
2. **TLS**: Use cert-manager or your preferred TLS solution
3. **Authentication**: Configure SSO (OIDC, OAuth, SAML) in the values

See `values.yaml` for configuration options.

## Post-Bootstrap: Automatic Discovery

Once bootstrapped, the ApplicationSet will:

1. **Discover repositories** with the `argocd` GitHub topic
2. **Generate Applications** automatically for each discovered repo
3. **Deploy content** from each repo's `argocd/` directory
4. **Keep everything in sync** with auto-sync and self-heal enabled

### Adding a New Deployment Repository

See README.md for detailed instructions on setting up new deployment repositories.

Quick summary:
1. Create `argocd/Chart.yaml` and `argocd/values.yaml` in your repo
2. Add the `argocd` topic on GitHub
3. Push to GitHub
4. ApplicationSet automatically discovers and deploys within minutes

## Updating ArgoCD Itself

### Automatic Updates (via Renovate)

Renovate automatically:
1. Monitors the official argo-cd Helm chart
2. Creates PRs with version updates (after 28-day stability window)
3. Auto-merges patch and minor versions
4. Requests review for major versions

The ApplicationSet detects changes and auto-syncs the update.

### Manual Updates

1. Edit `argocd/Chart.yaml` and update the `argo-cd` dependency version
2. Run `helm dependency update argocd/`
3. Commit and push:
   ```bash
   git add argocd/Chart.yaml argocd/Chart.lock
   git commit -m "chore: update argo-cd helm chart to X.Y.Z"
   git push
   ```
4. ArgoCD automatically syncs and applies the update

## Troubleshooting

### ApplicationSet not discovering repositories

Check if the ApplicationSet is running:
```bash
kubectl get applicationsets -n argocd
kubectl describe applicationset argocd-deployments -n argocd
```

Common issues:
- **GitHub token missing or invalid**: Verify the `github-token` secret exists
  ```bash
  kubectl get secret github-token -n argocd
  ```
- **Organization name mismatch**: Check that organization name in ApplicationSet matches your GitHub org
- **Repository topic not set**: Ensure repositories have the `argocd` topic on GitHub

Check ApplicationSet logs:
```bash
kubectl logs -n argocd deployment/argocd-applicationset-controller -f
```

### Generated Application is out of sync

Check the Application status:
```bash
kubectl get applications -n argocd
kubectl describe application <app-name> -n argocd
```

Manually sync:
```bash
kubectl patch application <app-name> -n argocd -p '{"status":{"operationState":{"finishedAt":null}}}' --type merge
```

### ArgoCD pods are stuck

Check the pod logs:
```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=50
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server --tail=50
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-repo-server --tail=50
```

### Repository not being discovered

Verify:
1. GitHub topic is set to `argocd` (not `argo-cd`)
2. Repository has `argocd/Chart.yaml` file
3. GitHub token has appropriate permissions
4. Organization name matches what's in ApplicationSet

### Chart dependency resolution fails

Ensure Chart.yaml dependencies are current:
```bash
cd argocd/
helm dependency update
cd ..
git add argocd/Chart.lock
git commit -m "chore: update chart dependencies"
git push
```

## Rollback

To rollback to a previous version:

1. Find the previous chart version in git history:
```bash
git log --oneline argocd/Chart.yaml
```

2. Revert the change:
```bash
git revert <commit-sha>
```

3. ArgoCD will automatically apply the rollback

## Clean Up / Uninstall

If you need to completely remove ArgoCD:

```bash
# Delete the Application (this triggers cleanup via finalizer)
kubectl delete application argocd -n argocd

# Wait for ArgoCD pods to be removed
kubectl wait --for=delete pod \
  -l app.kubernetes.io/part-of=argocd \
  -n argocd \
  --timeout=300s

# Delete the namespace
kubectl delete namespace argocd
```

## References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD Helm Chart](https://github.com/argoproj/argo-helm)
- [Renovate Documentation](https://docs.renovatebot.com/)
