# Production Kubernetes platform

This repository is the Git-authoritative production state for one K3s server. It contains plain Kubernetes/Kustomize resources and one GitHub Actions CD workflow; it does not use Argo CD, cert-manager, staging, or a public Kubernetes API.

## Layout

```text
base/                    Deployments, ClusterIP Services, Ingress, policies
overlays/production/     Domain configuration and immutable image digests
.github/workflows/       Hosted validation and local least-privilege CD
```

The namespace itself and `github-deployer` Role are created by immutable cloud-init. They are intentionally excluded from Kustomize because a namespaced deployment identity must not create namespaces or cluster-wide RBAC.

## Validate

```bash
kubectl kustomize overlays/production
```

Set `APP_DOMAIN` in `overlays/production/kustomization.yml` to the same value as Terraform's `domain`. Traefik performs global HTTP-to-HTTPS redirect and stores automatically renewed Let's Encrypt certificates on its local persistent volume.

## Automated deployment

The scheduled `Deploy production` workflow runs only on the `k3s-production` self-hosted runner. It authenticates to GHCR with the least-privilege built-in token, resolves every `production` tag to a digest, records changes in this repository, verifies Kubernetes authorization boundaries, applies the overlay, waits for four rollouts, and checks the public `/healthz` endpoint.

Pull requests and manifest validation always run on GitHub-hosted runners. Never add a `pull_request` trigger to a job using the production runner.

Images are immutable in the rendered manifests. The moving `production` tag is only a discovery pointer and is never deployed directly.

## Operations

```bash
kubectl -n production get deployments,pods,services,ingress
kubectl -n production rollout history deployment/web-frontend
kubectl -n production rollout undo deployment/web-frontend
```

After emergency rollback, revert the corresponding digest commit so Git remains authoritative. One replica minimizes resource usage but cannot guarantee zero downtime during node failure; the rolling strategy uses a surge pod and no intentional unavailable pod during normal updates.

Secret manifests with real values are forbidden. Add only `secret.example.yaml` placeholders if an application later needs runtime configuration.