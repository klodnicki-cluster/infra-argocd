# ArgoCD Self-Managed Repository

This repository contains the configuration for self-managed ArgoCD deployment. ArgoCD deploys and manages itself from this repository.

## Purpose

- **Auto-Discovered Deployments**: Repositories tagged with the `argocd` GitHub topic are automatically discovered and deployed
- **ApplicationSet-Driven**: Uses ArgoCD ApplicationSet for scalable, declarative repository management
- **GitOps-Driven Updates**: All updates to ArgoCD and deployment configurations are version-controlled in Git
- **Automated Updates**: Renovate automatically creates PRs for new ArgoCD/Helm chart versions
- **Scalable Pattern**: Add new repos, tag them with `argocd` topic → automatic deployment

## Repository Structure

```
.
├── README.md                                          # This file
├── BOOTSTRAP.md                                       # Initial setup guide
├── renovate.json                                      # Renovate configuration
└── argocd/
    ├── Chart.yaml                                     # Wrapper chart referencing argo-cd
    ├── Chart.lock                                     # Helm dependency lock file
    ├── values.yaml                                    # Helm values for argo-cd
    ├── templates/
    │   └── argocd-deployments-applicationset.yaml    # ApplicationSet for auto-discovery
    └── values/
        └── argocd-values.yaml                         # Detailed values reference
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

#### `argocd/templates/argocd-deployments-applicationset.yaml`
- ApplicationSet that discovers repositories with the `argocd` GitHub topic
- Generates Applications automatically for all discovered repositories
- Each Application deploys content from the repo's `argocd/` directory
- Namespace: repository name (sanitized for Kubernetes)

#### `renovate.json`
- Renovate bot configuration
- Monitors Helm chart versions with 28-day stability window
- Auto-merges minor and patch version updates
- Requires manual review for major version updates

## How It Works

### Application Auto-Discovery

The repository contains an ApplicationSet that automatically discovers all repositories in the GitHub organization tagged with the `argocd` topic:

```
GitHub Organization (klodnicki-cluster)
├── repo-1 (topic: argocd) → Auto-deployed
├── repo-2 (topic: argocd) → Auto-deployed
├── repo-3 (NO topic)      → Ignored
└── repo-4 (topic: argocd) → Auto-deployed
```

### Discovery & Deployment Flow

```
1. ApplicationSet polls GitHub for repos with 'argocd' topic
    ↓
2. For each discovered repo:
    - Checks for argocd/Chart.yaml
    - Reads argocd/values.yaml
    ↓
3. Generates Application for each repo
    - Name: {repo-name}
    - Namespace: {repo-name}
    - Release: {repo-name}
    ↓
4. ArgoCD deploys the Application
    - Auto-sync enabled
    - Self-heal enabled
    ↓
5. Application deploys the Helm chart from argocd/ directory
```

## Setting Up a New Deployment Repository

To add a new application/service that should be auto-deployed by ArgoCD:

### Step 1: Create the `argocd/` Directory

In your repository:

```bash
mkdir -p argocd/templates
```

### Step 2: Create `argocd/Chart.yaml`

Reference the Helm chart for your application:

```yaml
apiVersion: v2
name: my-app
dependencies:
  - name: my-app
    version: 1.0.0
    repository: https://charts.example.com
```

### Step 3: Create `argocd/values.yaml`

Configure your Helm chart:

```yaml
my-app:
  replicas: 2
  image:
    tag: v1.0.0
  # ... other values
```

### Step 4: Add the `argocd` Topic

In GitHub UI:
1. Go to your repository settings
2. Find "Topics" section
3. Add topic: `argocd`
4. Save

### Step 5: Push to GitHub

```bash
git add argocd/
git commit -m "feat: add argocd deployment configuration"
git push
```

### Step 6: Wait for Discovery

The ApplicationSet will discover your repository within a few minutes. Verify:

```bash
kubectl get applications -n argocd
# Should show your-repo-name in the list
```

Your application is now deployed to the `{repo-name}` namespace! 🎉

### Optional: Custom Resources

For advanced needs, add custom Kubernetes resources or Applications in `argocd/templates/`:

```yaml
# argocd/templates/custom-app.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app-custom
spec:
  # Custom application configuration
  ...
---
apiVersion: v1
kind: Secret
metadata:
  name: my-app-secrets
data:
  key: secret-value
```

These resources will be deployed alongside the default application.

## Updating ArgoCD Itself

### Automatic Updates (via Renovate)

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
6. ArgoCD detects change and auto-syncs
    ↓
7. ArgoCD updates itself automatically
```

### Manual Updates

1. Edit `argocd/Chart.yaml` and update the `argo-cd` dependency version
2. Run `helm dependency update argocd/` to update `Chart.lock`
3. Commit and push:
   ```bash
   git add argocd/Chart.yaml argocd/Chart.lock
   git commit -m "chore: update argo-cd helm chart to <new-version>"
   git push
   ```
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
4. Commit and push
5. ArgoCD automatically applies the update

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
