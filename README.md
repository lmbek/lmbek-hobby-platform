# Platform Kubernetes Manifests & GitOps State

This repository holds the declarative Kubernetes manifests and Kustomize overlays managed by ArgoCD.

---

## 📁 Manifest Organization

```text
├── cert-manager/           cert-manager v1.17.1 & Let's Encrypt ClusterIssuers (staging & prod)
├── base/                   Shared workload definitions (Deployments, Services, Ingress, Middlewares)
│   ├── websites/           web-frontend website (Port 8080)
│   ├── applications/       placeholder1 (8082) & placeholder2 (8081)
│   ├── docs/               docs portal (Port 80)
│   ├── middlewares.yml     Security headers, HTTPS redirect & rate limiting
│   ├── ingress.yml         Traefik path-based Ingress routing
│   └── kustomization.yml
├── overlays/
│   ├── staging/            Staging environment overlay (staging namespace, staging certs & hosts)
│   └── production/         Production environment overlay (production namespace, prod certs & hosts)
└── argocd/                 ArgoCD Application CRDs
    ├── staging.yml
    └── production.yml
```

---

## 🔒 Automated TLS Certificates & Proxies

- **cert-manager**: Automatically provisions and renews SSL/TLS certificates from Let's Encrypt using HTTP-01 challenges via Traefik.
- **ClusterIssuers**:
  - `letsencrypt-staging`: Testing solver to prevent Let's Encrypt rate limits.
  - `letsencrypt-prod`: Trusted production CA issuing valid certificates into `production-platform-tls`.
- **Traefik Middlewares**:
  - `redirect-to-https`: Automatic HTTP to HTTPS upgrade.
  - `security-headers`: HSTS, X-Content-Type-Options, X-Frame-Options, X-XSS-Protection.
  - `rate-limit`: Request burst and rate protection.

---

## 🌐 Ingress Routing Map

All services are exposed via Traefik Ingress on standard HTTP/HTTPS:
- `/` &rarr; `web-frontend:8080`
- `/service1` &rarr; `placeholder1-service:8082`
- `/service2` &rarr; `placeholder2-service:8081`
- `/docs` &rarr; `docs:80`

---

## 🚀 Promotion & Deployment Lifecycle

| Environment | Trigger | Image Tags | Namespace | Target URL |
|---|---|---|---|---|
| **Staging** | `git push` to `main` | `staging-latest`, `staging-<sha>` | `staging` | `https://staging.<your-ip>` |
| **Production** | GitHub Release published (`v*.*.*`) | `latest`, `v*.*.*`, `<sha>` | `production` | `https://<your-ip>` |

ArgoCD continuously monitors this platform repository and auto-reconciles workloads in both namespaces with zero downtime.

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
