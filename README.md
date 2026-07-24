# ArgoCD Self-Managed Repository

This repository contains the configuration for self-managed ArgoCD deployment. ArgoCD deploys and manages itself from this repository.

## Purpose

- **Self-Deployment**: ArgoCD uses a self-referential Application to deploy and update itself
- **GitOps-Driven Updates**: All updates to ArgoCD configuration are version-controlled in Git
- **Automated Updates**: Renovate automatically creates PRs for new ArgoCD/Helm chart versions
- **Single Source of Truth**: This repository becomes the definitive source for ArgoCD configuration

## Repository Structure

```
.
├── README.md                              # This file
├── BOOTSTRAP.md                           # Initial setup guide
├── renovate.json                          # Renovate configuration
└── argocd/
    ├── Chart.yaml                         # Wrapper chart referencing argo-cd
    ├── Chart.lock                         # Helm dependency lock file
    ├── values.yaml                        # Helm values for argo-cd
    ├── templates/
    │   └── argocd-application.yaml       # Self-referential Application manifest
    └── values/
        └── argocd-values.yaml            # Detailed values configuration (reference)
```

### Key Files

#### `argocd/Chart.yaml`
- Wrapper Helm chart that dependencies on the official `argo-cd` chart
- Renovate monitors this file for chart version updates
- Chart version is pinned for reproducible deployments

#### `argocd/values.yaml`
- Helm values passed to the argo-cd chart
- Contains configuration for ArgoCD components (server, controller, repo-server, etc.)
- Resource limits, persistence, RBAC, and security settings

#### `argocd/applications/argocd-application.yaml`
- Self-referential ArgoCD Application manifest
- Deploys ArgoCD from this repository
- Configured for auto-sync with self-healing enabled
- Auto-prunes resources no longer in the repository

#### `renovate.json`
- Renovate bot configuration
- Monitors Helm chart versions with 28-day stability window
- Auto-merges minor and patch version updates
- Requires manual review for major version updates

## How It Works

### Initial Bootstrap

1. Install ArgoCD manually on the cluster using the Helm chart (see [BOOTSTRAP.md](BOOTSTRAP.md))
2. Apply the Application manifest: `argocd-application.yaml`
3. ArgoCD is now self-managing

### Self-Management Cycle

```
1. Changes pushed to this repo
    ↓
2. ArgoCD detects changes via Application webhook/polling
    ↓
3. ArgoCD reconciles cluster state with repository
    ↓
4. New ArgoCD version deployed automatically
```

### Update Workflow

#### Automatic Updates (via Renovate)

```
1. Renovate checks for new argo-cd chart versions daily
    ↓
2. After 28-day stability window:
    ↓
3. Renovate creates PR with version bump
    ↓
4. (Auto-merge for patch/minor, manual review for major)
    ↓
5. PR merged to main branch
    ↓
6. ArgoCD detects change in this repo
    ↓
7. ArgoCD applies the update automatically
```

#### Manual Updates

1. Edit `argocd/Chart.yaml` and update the `argo-cd` dependency version
2. Run `helm dependency update` to update `Chart.lock`
3. Commit and push changes
4. ArgoCD automatically applies the update

## Configuration

### Customizing ArgoCD

Edit `argocd/values.yaml` to customize:
- Domain/URL
- RBAC policies
- Resource limits
- Persistence settings
- Notification/monitoring configuration
- SSO/authentication

All changes are automatically applied by ArgoCD's auto-sync policy.

### Updating Chart Version

To update the ArgoCD Helm chart version:

1. Edit `argocd/Chart.yaml`
2. Update the `version` field in the dependencies section
3. Run `helm dependency update argocd/` to update `Chart.lock`
4. Commit and push:
   ```bash
   git add argocd/Chart.yaml argocd/Chart.lock
   git commit -m "chore: update argo-cd helm chart to <new-version>"
   git push
   ```

Or let Renovate do it automatically!

## Renovate Configuration Details

### Safety Windows
- **Stability Period**: 28 days (4 weeks) before suggesting chart updates
- **Minimum Release Age**: 28 days to ensure stability
- **Concurrent PRs**: Limited to 2 to avoid overwhelming the repo

### Auto-Merge Policy
- ✅ **Patch versions** (e.g., 7.3.3 → 7.3.4): Auto-merged
- ✅ **Minor versions** (e.g., 7.3.0 → 7.4.0): Auto-merged
- ❌ **Major versions** (e.g., 7.x.x → 8.0.0): Requires manual review

### Labels
- `renovate`: Applied to all Renovate PRs
- `dependencies`: Indicates dependency update
- `helm-major-update`: Applied to major version PRs requiring review

## Monitoring & Troubleshooting

### Check Application Status

```bash
# Get application status
kubectl get application argocd -n argocd

# Get detailed information
kubectl describe application argocd -n argocd

# Watch for syncs
kubectl get application argocd -n argocd -w

# Check sync history
kubectl get application argocd -n argocd -o jsonpath='{.status.operationState}'
```

### View Logs

```bash
# ArgoCD application controller logs
kubectl logs -n argocd deployment/argocd-application-controller -f

# ArgoCD server logs
kubectl logs -n argocd deployment/argocd-server -f

# ArgoCD repo-server logs
kubectl logs -n argocd deployment/argocd-repo-server -f
```

### Common Issues

- **Out of Sync**: Check `argocd/values.yaml` syntax and ensure Chart.yaml dependencies are current
- **Sync Fails**: Review logs above and check for resource conflicts
- **Renovate PRs Not Creating**: Verify renovate.json is valid and Renovate is enabled on the repo

## Getting Started

See [BOOTSTRAP.md](BOOTSTRAP.md) for initial setup instructions.

## Useful Commands

```bash
# Check all ArgoCD resources
kubectl get all -n argocd

# Access ArgoCD UI (port-forward)
kubectl port-forward svc/argocd-server -n argocd 8080:443

# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d

# Manually sync the ArgoCD application
kubectl patch application argocd -n argocd --type merge -p '{"status":{"operationState":{"finishedAt":null}}}'

# View recent PRs from Renovate
gh pr list --author renovate

# Check Renovate logs
gh run list --workflow=renovate
```

## Contributing

When making manual changes to this repository:

1. Create a feature branch
2. Make changes to `argocd/values.yaml` or other configuration files
3. Test locally if possible (see BOOTSTRAP.md)
4. Create a PR
5. Once merged, ArgoCD automatically applies the changes

## References

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ArgoCD Helm Chart Repository](https://github.com/argoproj/argo-helm)
- [Renovate Documentation](https://docs.renovatebot.com/)
- [Kubernetes GitOps](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/#declarative-object-configuration)

## License

See LICENSE file for details.
