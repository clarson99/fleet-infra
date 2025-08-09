# Fleet Infrastructure

A GitOps repository for managing Kubernetes clusters using Flux CD. This repository follows Flux best practices for multi-cluster fleet management and provides a structured approach to deploying infrastructure and applications across different environments.

## About This Project

This fleet-infra repository serves as the central source of truth for Kubernetes cluster configurations using GitOps principles. It manages:

- **Multiple Kubernetes clusters** across different environments (production, staging, development, local)
- **Infrastructure components** like ingress controllers, cert-manager, monitoring, and other cluster-wide services
- **Application deployments** with environment-specific configurations and overlays
- **Flux CD configurations** for automated synchronization and deployment

The repository structure follows Flux community best practices, ensuring scalability, maintainability, and security across your Kubernetes fleet.

### Key Features

- 🔄 **GitOps Workflow**: All changes are version-controlled and automatically deployed
- 🏗️ **Infrastructure as Code**: Cluster infrastructure managed declaratively
- 🚀 **Multi-Environment Support**: Separate configurations for different environments
- 🔒 **Security Best Practices**: RBAC, secret management, and secure deployments
- 📊 **Observability**: Built-in monitoring and GitOps dashboard integration
- 🔧 **Extensible**: Easy to add new clusters and applications

## Install Prerequisites

Before working with this repository, ensure you have the following tools installed:

### Required Tools

1. **Kubernetes CLI (kubectl)**
   ```bash
   # macOS
   brew install kubectl
   
   # Linux
   curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
   sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
   ```

2. **Flux CLI**
   ```bash
   # macOS
   brew install fluxcd/tap/flux
   
   # Linux
   curl -s https://fluxcd.io/install.sh | sudo bash
   
   # Verify installation
   flux --version
   ```

3. **Colima** (Container runtime for local development)
   ```bash
   # macOS
   brew install colima
   
   # Linux
   curl -LO https://github.com/abiosoft/colima/releases/latest/download/colima-Linux-x86_64
   sudo install colima-Linux-x86_64 /usr/local/bin/colima
   ```

4. **Docker CLI** (for container operations)
   ```bash
   # macOS
   brew install docker
   
   # Linux - follow Docker CLI installation guide for your distribution
   ```

### Optional Tools

- **k9s** - Terminal-based Kubernetes dashboard
  ```bash
  brew install k9s  # macOS
  ```

- **kubectx/kubens** - Context and namespace switching
  ```bash
  brew install kubectx  # macOS
  ```

## Testing Against Local Cluster

Use Colima with Kubernetes for local testing of configurations before applying to development or production environments.

### 1. Start Colima with Kubernetes

```bash
# Start Colima with Kubernetes enabled
colima start --kubernetes

# Verify Colima is running
colima status

# Verify Kubernetes cluster is available
kubectl cluster-info
```

### 2. Bootstrap Flux on Local Cluster

```bash
# Set your GitHub username and repository
export GITHUB_USER=your-username
export GITHUB_REPO=fleet-infra

# Bootstrap Flux (this will create the clusters/local directory if it doesn't exist)
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=$GITHUB_REPO \
  --branch=main \
  --path=./clusters/local \
  --personal
```

### 3. Verify Flux Installation

```bash
# Check Flux components
flux get all

# Check kustomizations
flux get kustomizations

# Check sources
flux get sources git
```

### 4. Monitor Deployments

```bash
# Watch all Flux resources
flux get all --watch

# Check specific kustomization
flux get kustomization flux-system

# View logs
flux logs --follow --tail=10
```

### 5. Clean Up Local Environment

```bash
# Stop Colima when done
colima stop

# Or delete the Colima instance completely
colima delete
```

## Working with Dev Cluster

The development cluster is used for testing changes before promoting to staging and production.

### 1. Connect to Dev Cluster

```bash
# Set kubectl context to dev cluster
kubectl config use-context dev-cluster-context

# Verify connection
kubectl get nodes
```

### 2. Bootstrap Dev Cluster (if not already done)

```bash
# Bootstrap Flux on dev cluster
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=$GITHUB_REPO \
  --branch=main \
  --path=./clusters/development \
  --personal
```

### 3. Deploy Changes to Dev

1. **Make changes** to configurations in your local repository
2. **Test locally** using the local Colima cluster
3. **Commit and push** changes to the main branch
4. **Monitor deployment** in dev cluster:
   ```bash
   # Force reconciliation if needed
   flux reconcile kustomization flux-system
   
   # Watch deployment progress
   flux get kustomizations --watch
   ```

### 4. Validate Deployments

```bash
# Check application status
kubectl get pods -A

# Check Flux status
flux get all

# View application logs
kubectl logs -f deployment/app-name -n namespace
```

## Adding a Cluster

To add a new cluster to the fleet, follow these steps:

### 1. Create Cluster Directory Structure

```bash
# Create new cluster directory
mkdir -p clusters/new-cluster-name/{flux-system,infrastructure,apps}

# Create kustomization files
touch clusters/new-cluster-name/infrastructure/kustomization.yaml
touch clusters/new-cluster-name/apps/kustomization.yaml
```

### 2. Configure Infrastructure Kustomization

Create `clusters/new-cluster-name/infrastructure/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../infrastructure/controllers
  - ../../infrastructure/configs
  - ../../infrastructure/sources
```

### 3. Configure Apps Kustomization

Create `clusters/new-cluster-name/apps/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../apps/base
# Add environment-specific patches here
```

### 4. Bootstrap New Cluster

```bash
# Connect to the new cluster
kubectl config use-context new-cluster-context

# Bootstrap Flux
flux bootstrap github \
  --owner=$GITHUB_USER \
  --repository=$GITHUB_REPO \
  --branch=main \
  --path=./clusters/new-cluster-name \
  --personal
```

### 5. Verify Cluster Setup

```bash
# Check Flux installation
flux get all

# Verify infrastructure deployment
kubectl get pods -n flux-system

# Check application deployments
kubectl get pods -A
```

## Adding an Application to a Cluster

Follow these steps to add a new application to your fleet:

### 1. Create Base Application Configuration

Create the base application directory and manifests:

```bash
# Create application directory
mkdir -p apps/base/my-app

# Create base kustomization
cat > apps/base/my-app/kustomization.yaml << EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - deployment.yaml
  - service.yaml
  - ingress.yaml
EOF
```

### 2. Add Application Manifests

Create your application manifests in `apps/base/my-app/`:

- `deployment.yaml` - Kubernetes Deployment
- `service.yaml` - Kubernetes Service  
- `ingress.yaml` - Ingress configuration (if needed)

### 3. Create Flux Source (if external)

If your application comes from an external Git repository:

```yaml
# apps/base/my-app/source.yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 1m0s
  ref:
    branch: main
  url: https://github.com/your-org/my-app
```

### 4. Create Flux Kustomization

```yaml
# apps/base/my-app/kustomization-flux.yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: my-app
  namespace: flux-system
spec:
  interval: 5m0s
  path: "./deploy"  # or wherever your manifests are in the source repo
  prune: true
  sourceRef:
    kind: GitRepository
    name: my-app
  targetNamespace: my-app
  dependsOn:
    - name: infrastructure
```

### 5. Update Base Apps Kustomization

Add your new application to `apps/base/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - podinfo
  - weave-gitops
  - my-app  # Add your new app here
```

### 6. Add Environment-Specific Overlays (Optional)

Create environment-specific configurations:

```bash
# Create overlay for production
mkdir -p apps/production/my-app

cat > apps/production/my-app/kustomization.yaml << EOF
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../base/my-app
patchesStrategicMerge:
  - deployment-patch.yaml
EOF

# Create production-specific patches
cat > apps/production/my-app/deployment-patch.yaml << EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  template:
    spec:
      containers:
      - name: my-app
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
EOF
```

### 7. Deploy to Cluster

1. **Commit and push** your changes:
   ```bash
   git add .
   git commit -m "Add my-app application"
   git push origin main
   ```

2. **Monitor deployment**:
   ```bash
   # Watch Flux reconciliation
   flux get kustomizations --watch
   
   # Check application pods
   kubectl get pods -n my-app
   
   # Force reconciliation if needed
   flux reconcile kustomization apps
   ```

### 8. Verify Application

```bash
# Check application status
kubectl get all -n my-app

# Check application logs
kubectl logs -f deployment/my-app -n my-app

# Test application (if it has a service)
kubectl port-forward svc/my-app 8080:80 -n my-app
```

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

### Colima-Specific Issues

```bash
# Check Colima status
colima status

# Restart Colima if needed
colima restart

# Check Kubernetes context
kubectl config current-context

# Should show: colima
```

## Resources

- [Flux Documentation](https://fluxcd.io/docs/)
- [Flux Best Practices](https://fluxcd.io/docs/guides/repository-structure/)
- [Kustomize Documentation](https://kustomize.io/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Colima Documentation](https://github.com/abiosoft/colima)
- [PROMPT.md](./PROMPT.md) - Detailed project structure and best practices
