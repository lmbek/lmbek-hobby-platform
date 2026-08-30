# Platform Kubernetes Manifests & GitOps State

This repository holds the declarative Kubernetes manifests and Kustomize overlays managed by ArgoCD.

---

## 📁 Manifest Organization

```text
├── cert-manager/           Trusted Let's Encrypt ClusterIssuer
├── base/                   Shared workload definitions (Deployments, Services, Ingress, Middlewares)
│   ├── websites/           web-frontend website (Port 8080)
│   ├── applications/       placeholder1 (8082) & placeholder2 (8081)
│   ├── docs/               docs portal (Port 80)
│   ├── middlewares.yml     Security headers, HTTPS redirect & rate limiting
│   ├── ingress.yml         Traefik path-based Ingress routing
│   └── kustomization.yml
├── overlays/
│   ├── staging/            Internal staging namespace and image digests
│   └── production/         Production namespace, hosts, certificate, and image digests
└── argocd/                 ArgoCD Application CRDs
    ├── staging.yml
    └── production.yml
```

---

## 🔒 Automated TLS Certificates & Proxies

- **cert-manager**: Automatically provisions and renews SSL/TLS certificates from Let's Encrypt using HTTP-01 challenges via Traefik.
- **ClusterIssuer**: `letsencrypt-prod` issues the trusted production certificate.
- **Traefik Middlewares**:
  - `redirect-to-https`: Automatic HTTP to HTTPS upgrade.
  - `security-headers`: HSTS, X-Content-Type-Options, X-Frame-Options, X-XSS-Protection.
  - `rate-limit`: Request burst and rate protection.

---

## 🌐 Public Ingress

Only `https://lmbek.dk` is public and routes to `web-frontend:8080`. The
documentation, placeholder services, and complete staging environment have no
Ingress and are reachable only by workloads inside their Kubernetes namespace.

---

## 🚀 Promotion & Deployment Lifecycle

| Environment | Image Reference | Namespace | Target URL |
|---|---|---|---|
| **Staging** | Immutable `sha256` digest | `staging` | Internal only |
| **Production** | Immutable `sha256` digest | `production` | `https://lmbek.dk` |

Service pushes build `staging-latest` images. Deploy one by resolving its registry
digest, updating `overlays/staging/kustomization.yml`, and merging that declarative
change. Promote the tested digest by copying it to
`overlays/production/kustomization.yml`. Argo CD reconciles both namespaces from Git.

Before promoting, verify each referenced image from your local environment:

```bash
docker buildx imagetools inspect ghcr.io/lmbek/lmbek-hobby-web-frontend:staging-latest
docker buildx imagetools inspect ghcr.io/lmbek/lmbek-hobby-placeholder1-service:staging-latest
docker buildx imagetools inspect ghcr.io/lmbek/lmbek-hobby-placeholder2-service:staging-latest
docker buildx imagetools inspect ghcr.io/lmbek/lmbek-hobby-docs:staging-latest
```

Use the top-level `Digest` value for each image. Digest pinning makes production
repeatable and causes an explicit Deployment rollout whenever a promoted digest changes.

---

## 🛠️ Testing Manifests Locally

You can render and validate Kustomize overlays using `kubectl`:

```bash
# Render Base:
kubectl kustomize base

# Render Staging Overlay:
kubectl kustomize overlays/staging

# Render Production Overlay:
kubectl kustomize overlays/production
```
