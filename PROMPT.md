# Fleet Infrastructure - Flux GitOps Repository

This repository contains Flux configurations for managing Kubernetes clusters using GitOps principles. The structure follows Flux best practices for multi-cluster fleet management.

## Repository Structure

```
fleet-infra/
├── clusters/                    # Cluster-specific configurations
│   ├── production/             # Production cluster configs
│   ├── staging/                # Staging cluster configs
│   ├── development/            # Development cluster configs
│   └── local/                  # Local testing cluster configs
│       ├── flux-system/        # Flux system components
│       │   ├── gotk-components.yaml
│       │   ├── gotk-sync.yaml
│       │   └── kustomization.yaml
│       ├── infrastructure/     # Infrastructure components
│       │   ├── controllers/    # Controllers (ingress, cert-manager, etc.)
│       │   ├── configs/        # ConfigMaps, Secrets (non-sensitive)
│       │   └── kustomization.yaml
│       └── apps/              # Application deployments
│           ├── base/          # Base application configs
│           ├── production/    # Production-specific overlays
│           ├── staging/       # Staging-specific overlays
│           └── kustomization.yaml
├── infrastructure/             # Shared infrastructure components
│   ├── controllers/           # Cluster controllers
│   │   ├── ingress-nginx/
│   │   ├── cert-manager/
│   │   ├── external-dns/
│   │   └── kustomization.yaml
│   ├── configs/              # Shared configurations
│   │   ├── cluster-issuers/
│   │   ├── network-policies/
│   │   └── kustomization.yaml
│   └── sources/              # Shared Flux sources
│       ├── helm-repositories.yaml
│       ├── git-repositories.yaml
│       └── kustomization.yaml
├── apps/                      # Application definitions
│   ├── base/                 # Base application configurations
│   │   ├── podinfo/
│   │   ├── weave-gitops/
│   │   └── kustomization.yaml
│   ├── production/           # Production overlays
│   ├── staging/              # Staging overlays
│   └── development/          # Development overlays
├── tenants/                  # Multi-tenancy configurations (if applicable)
│   ├── team-a/
│   ├── team-b/
│   └── kustomization.yaml
└── scripts/                  # Utility scripts
    ├── bootstrap.sh
    ├── check-prerequisites.sh
    └── generate-secrets.sh
```

## Flux Best Practices

### 1. Repository Organization

- **Separation of Concerns**: Keep infrastructure, applications, and cluster-specific configs separate
- **Environment Promotion**: Use overlays for environment-specific configurations
- **Shared Resources**: Place common configurations in shared directories
- **Tenant Isolation**: Use separate directories for multi-tenant setups

### 2. Directory Structure Guidelines

#### Clusters Directory
- Each cluster has its own subdirectory under `clusters/`
- Contains cluster-specific Flux system configuration
- References shared infrastructure and applications
- Environment-specific overlays and patches

#### Infrastructure Directory
- Shared infrastructure components across clusters
- Controllers (ingress, cert-manager, monitoring)
- Cluster-wide configurations
- Helm repositories and Git sources

#### Apps Directory
- Application base configurations
- Environment-specific overlays
- Kustomization files for each environment

### 3. Kustomization Structure

Each directory should contain a `kustomization.yaml` file that:
- Lists all resources in the directory
- Defines dependencies between components
- Specifies patches and overlays
- Sets appropriate intervals for reconciliation

### 4. Naming Conventions

- Use kebab-case for directory and file names
- Prefix cluster-specific resources with cluster name when needed
- Use descriptive names for Flux resources
- Follow semantic versioning for application versions

### 5. Security Best Practices

- Store sensitive data in Kubernetes secrets (not in Git)
- Use Flux's secret management features
- Implement RBAC for Flux controllers
- Use separate Git repositories for sensitive configurations if needed

### 6. Flux Sources

#### Git Repositories
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: fleet-infra
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  url: https://github.com/your-org/fleet-infra
```

#### Helm Repositories
```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: ingress-nginx
  namespace: flux-system
spec:
  interval: 24h
  url: https://kubernetes.github.io/ingress-nginx
```

### 7. Kustomization Resources

#### Infrastructure Kustomization
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: infrastructure
  namespace: flux-system
spec:
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: "./infrastructure"
  prune: true
  wait: true
```

#### Application Kustomization
```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: apps
  namespace: flux-system
spec:
  dependsOn:
    - name: infrastructure
  interval: 10m0s
  sourceRef:
    kind: GitRepository
    name: fleet-infra
  path: "./apps/production"
  prune: true
```

### 8. Dependency Management

- Infrastructure components should be deployed before applications
- Use `dependsOn` field in Kustomizations to define dependencies
- Implement health checks and readiness probes
- Use `wait: true` for critical infrastructure components

### 9. Monitoring and Observability

- Deploy monitoring stack as part of infrastructure
- Use Flux's built-in metrics and alerts
- Implement GitOps dashboard (Weave GitOps)
- Monitor Git repository sync status

### 10. Disaster Recovery

- Backup Flux configurations
- Document bootstrap procedures
- Test cluster recreation from Git
- Maintain infrastructure as code principles

## Getting Started

### Prerequisites

- Kubernetes cluster (v1.21+)
- kubectl configured
- Flux CLI installed
- Git repository access

### Bootstrap Process

1. **Install Flux CLI**:
   ```bash
   curl -s https://fluxcd.io/install.sh | sudo bash
   ```

2. **Bootstrap Flux**:
   ```bash
   flux bootstrap github \
     --owner=your-org \
     --repository=fleet-infra \
     --branch=main \
     --path=./clusters/my-cluster
   ```

3. **Verify Installation**:
   ```bash
   flux get kustomizations
   flux get sources git
   ```

### Adding New Applications

1. Create base configuration in `apps/base/`
2. Add environment-specific overlays
3. Update cluster kustomization files
4. Commit and push changes
5. Monitor deployment with `flux get kustomizations`

### Adding New Clusters

1. Create new directory under `clusters/`
2. Copy flux-system configuration
3. Update Git repository references
4. Bootstrap new cluster
5. Add cluster-specific configurations

## Troubleshooting

### Common Commands

```bash
# Check Flux status
flux get all

# Check specific kustomization
flux get kustomizations -A

# Suspend/resume reconciliation
flux suspend kustomization <name>
flux resume kustomization <name>

# Force reconciliation
flux reconcile kustomization <name>

# Check logs
flux logs --follow --tail=10
```

### Common Issues

1. **Sync Failures**: Check Git repository access and branch references
2. **Resource Conflicts**: Verify resource ownership and dependencies
3. **RBAC Issues**: Ensure proper service account permissions
4. **Network Issues**: Check cluster connectivity to Git repositories

## Contributing

1. Follow the established directory structure
2. Test changes in development environment first
3. Use descriptive commit messages
4. Update documentation for new components
5. Follow Flux and Kubernetes best practices

## Resources

- [Flux Documentation](https://fluxcd.io/docs/)
- [Flux Best Practices](https://fluxcd.io/docs/guides/repository-structure/)
- [Kustomize Documentation](https://kustomize.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
