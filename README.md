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
│   ├── staging/            Staging namespace, HTTPS ingress, certificate, and image digests
│   └── production/         Production namespace, hosts, certificate, and image digests
└── argocd/                 ArgoCD Application CRDs
    ├── staging.yml
    └── production.yml
```

---

## 🔒 Automated TLS Certificates & Proxies

- **cert-manager**: Automatically provisions and renews SSL/TLS certificates from Let's Encrypt using HTTP-01 challenges via Traefik.
- **ClusterIssuer**: `letsencrypt-prod` issues trusted certificates for production and staging.
- **Traefik Middlewares**:
  - `redirect-to-https`: Automatic HTTP to HTTPS upgrade.
  - `security-headers`: HSTS, X-Content-Type-Options, X-Frame-Options, X-XSS-Protection.
  - `rate-limit`: Request burst and rate protection.

---

## 🌐 Public Ingress

`https://lmbek.dk` and `https://staging.lmbek.dk` route to their environment's
`web-frontend:8080`. The documentation and placeholder services remain internal.
Local development is HTTP-only and is configured separately in `local-orchestrator`.

---

## 🚀 Promotion & Deployment Lifecycle

| Environment | Image Reference | Namespace | Target URL |
|---|---|---|---|
| **Staging** | Immutable `sha256` digest | `staging` | `https://staging.lmbek.dk` |
| **Production** | Immutable `sha256` digest | `production` | `https://lmbek.dk` |

Service pushes to `main` publish images to GHCR. Every five minutes, this repository's
automated release workflow reads the public `staging-latest` manifest digests, updates
and auto-merges `overlays/staging/kustomization.yml`, waits for the public staging
health endpoint, and auto-merges the exact tested digests into
`overlays/production/kustomization.yml`. Argo CD reconciles both namespaces from Git.

The workflow uses the platform repository's built-in `GITHUB_TOKEN`; no
`PLATFORM_REPOSITORY_TOKEN` or service-repository secrets are required. Enable
repository auto-merge once so the automated pull requests can merge. It never
accesses the cluster or production servers.

For troubleshooting only, inspect referenced images from your local environment:

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

Pull requests and pushes to `main` run the same production and staging renders in
`.github/workflows/validate.yml`. This workflow validates manifests only; it never
connects to the cluster or deploys anything. Argo CD is the CD system and watches
`main` after a change is merged.
