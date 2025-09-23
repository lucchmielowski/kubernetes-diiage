# WordPress Kubernetes Manifests with Kustomize

This directory contains a Kustomize-based structure for deploying WordPress applications across multiple environments.

## Structure

```
manifests/
├── base/                          # Base resources shared across all environments
│   ├── kustomization.yaml        # Base kustomization configuration
│   ├── 00-deploy.yaml           # Base deployment (WordPress + MySQL)
│   ├── 00-pvc.yaml              # Persistent Volume Claims
│   └── 00-service.yaml          # Services
└── overlays/                     # Environment-specific configurations
    ├── env-00/                  # Environment 0 (Base)
    ├── env-01/                  # Environment 1 (With volumes)
    ├── env-02/                  # Environment 2 (With secrets)
    ├── env-03/                  # Environment 3 (With ConfigMap + Ingress)
    └── env-04/                  # Environment 4 (StatefulSet + Headless Service)
```

## Environments

### Base (env-00)
- Basic WordPress deployment with MySQL
- Default configuration
- Uses hardcoded passwords

### Environment 1 (env-01)
- Adds persistent volumes for WordPress
- Enhanced storage configuration
- Namespace: `wordpress-01`

### Environment 2 (env-02)
- Uses Kubernetes secrets for MySQL password
- More secure configuration
- Namespace: `wordpress-02`

### Environment 3 (env-03)
- Includes ConfigMap for MySQL configuration
- Adds Ingress for external access
- Enhanced MySQL configuration
- Namespace: `wordpress-03`

### Environment 4 (env-04)
- Uses StatefulSet instead of Deployment
- Headless service configuration
- More suitable for stateful workloads
- Namespace: `wordpress-04`

## Usage

### Build a specific environment:

```bash
# Build base configuration
kubectl kustomize base

# Build environment 1
kubectl kustomize overlays/env-01

# Build environment 3
kubectl kustomize overlays/env-03
```

### Apply to cluster:

```bash
# Apply environment 1
kubectl apply -k overlays/env-01

# Apply environment 3
kubectl apply -k overlays/env-03
```

### Delete from cluster:

```bash
# Delete environment 1
kubectl delete -k overlays/env-01
```

## Features

- **Environment Isolation**: Each environment uses separate namespaces
- **Resource Naming**: Automatic name prefixing (e.g., `env-01-wordpress`)
- **Label Management**: Consistent labeling across environments
- **Patch-based Configuration**: Environment-specific modifications using patches
- **Modern Kustomize Syntax**: Uses current `patches` and `labels` syntax

## Customization

To add a new environment:

1. Create a new directory under `overlays/` (e.g., `overlays/env-05/`)
2. Create a `kustomization.yaml` file referencing the base
3. Add any environment-specific patches or resources
4. Test with `kubectl kustomize overlays/env-05`
