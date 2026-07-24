# ArgoCD Bootstrap Guide

This guide explains how to initially set up ArgoCD on your Kubernetes cluster using this repository. After the initial bootstrap, ArgoCD will manage itself through the Application manifest.

## Prerequisites

- Kubernetes cluster (1.24+)
- `kubectl` configured to access your cluster
- `helm` CLI (3.10+)
- Git access to this repository

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

### Step 4: Wait for ArgoCD to Be Ready

```bash
kubectl wait --for=condition=Progressing=True application/argocd \
  --namespace argocd \
  --timeout=300s
```

Or watch the deployment:

```bash
kubectl rollout status deployment/argocd-application-controller -n argocd
kubectl rollout status deployment/argocd-server -n argocd
kubectl rollout status deployment/argocd-repo-server -n argocd
```

### Step 5: Apply the Self-Managing Application Manifest

Once ArgoCD is running, apply the Application manifest that will manage ArgoCD itself:

```bash
kubectl apply -f argocd/applications/argocd-application.yaml
```

### Step 6: Verify the Application

```bash
kubectl get applications -n argocd
kubectl describe application argocd -n argocd
```

You should see the application sync successfully.

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

## Post-Bootstrap: Automatic Updates

Once bootstrapped, the ArgoCD Application will:

1. **Monitor this repository** for changes
2. **Auto-sync** when changes are detected (Helm chart updates)
3. **Renovate** will automatically create PRs when:
   - A new argo-cd Helm chart version is available (after 28-day stability window)
   - Non-major version bumps will auto-merge
   - Major version bumps require manual review

## Updating ArgoCD

To update ArgoCD after bootstrap:

1. **For automatic updates**: Renovate will create a PR with the updated chart version
2. **For manual updates**: Edit `argocd/Chart.yaml` and update the version in the dependencies section
3. Push changes to this repository
4. ArgoCD will automatically sync and apply the update

Example of updating manually:

```bash
# Edit Chart.yaml and update the argo-cd dependency version
# Then push to git
git add argocd/Chart.yaml
git commit -m "chore: update argo-cd helm chart to 7.4.0"
git push

# ArgoCD will automatically detect and apply the change
kubectl get applications -n argocd  # Watch the sync
```

## Troubleshooting

### Application is out of sync

Check what's different:
```bash
kubectl describe application argocd -n argocd
```

Manually sync:
```bash
kubectl patch application argocd -n argocd -p '{"status":{"operationState":{"finishedAt":null}}}' --type merge
```

### ArgoCD pods are stuck

Check the pod logs:
```bash
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-application-controller --tail=50
kubectl logs -n argocd -l app.kubernetes.io/name=argocd-server --tail=50
```

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
