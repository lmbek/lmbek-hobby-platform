# Platform — Kubernetes & ArgoCD

This directory contains the Kubernetes manifests and ArgoCD application definitions for **staging** and **production** deployments.

Local development uses Docker Compose and lives in the [orchestrators](../../orchestrators) repository.

## Structure

```
platform/
├── argocd/                        # ArgoCD Application CRDs
│   ├── staging.yml                # Apps syncing staging overlays
│   └── production.yml             # Apps syncing production overlays
├── base/                          # Shared Kubernetes manifests
│   ├── applications/              # Deployments + Services for app services
│   │   ├── deployment.yml
│   │   ├── service.yml
│   │   └── kustomization.yml
│   └── docs/                      # Deployment + Service for docs
│       ├── deployment.yml
│       ├── service.yml
│       └── kustomization.yml
└── overlays/                      # Environment-specific overrides (Kustomize)
    ├── staging/
    │   ├── applications/
    │   │   └── kustomization.yml  # Sets namespace=staging, image tag=stage
    │   └── docs/
    │       └── kustomization.yml
    └── production/
        ├── applications/
        │   └── kustomization.yml  # Sets namespace=production, image tag=latest
        └── docs/
            └── kustomization.yml
```

## How it works

1. **Base manifests** define Deployments and Services with sensible defaults.
2. **Overlays** use Kustomize to set the namespace and image tags per environment.
3. **ArgoCD Applications** point to the overlay paths and automatically sync changes from Git.

## Adding a new service

1. Add a Deployment and Service to `base/applications/` (or create a new base directory).
2. Reference it in the base `kustomization.yml`.
3. Update the overlay `kustomization.yml` files to set the correct image tag.
4. ArgoCD will automatically pick up the changes on the next sync.

## Prerequisites

- A Kubernetes cluster with ArgoCD installed.
- Update the `repoURL` in the ArgoCD manifests to match your actual Git repository URL.
- The base manifests use `ghcr.io/lmbek` (GitHub Container Registry). Update if your organisation or registry differs.
