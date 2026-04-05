# deployments

GitOps manifests repository. Managed by ArgoCD.
All Kubernetes resources for all projects live here.

## Structure

```
deployments/
  pesaloop/                        ← PesaLoop project
    base/backend/
      deployment.yaml              ← Deployment + Service + ConfigMap + PDB
      secrets.yaml                 ← Template only — never commit real values
      kustomization.yaml
    overlays/
      dev/kustomization.yaml       ← 1 replica, debug logging, sandbox APIs
      staging/kustomization.yaml   ← 2 replicas, production-like, sandbox APIs
      production/kustomization.yaml← 3 replicas, live APIs, manual sync only
  argocd-apps/
    root-app.yaml                  ← apply this once to bootstrap everything
    pesaloop-dev.yaml
    pesaloop-staging.yaml
    pesaloop-production.yaml
```

## First time setup

```bash
# 1. Bootstrap ArgoCD (run once)
kubectl apply -f argocd-apps/root-app.yaml

# 2. Create the image pull secret for ghcr.io
kubectl create secret docker-registry ghcr-secret \
  --docker-server=ghcr.io \
  --docker-username=francis-mwas \
  --docker-password=YOUR_GITHUB_PAT \
  -n pesaloop-dev

# 3. Apply secrets for dev (never commit the plain file)
cp pesaloop/base/backend/secrets.yaml /tmp/pesaloop-secrets.yaml
# Fill in real values in /tmp/pesaloop-secrets.yaml
kubectl apply -f /tmp/pesaloop-secrets.yaml -n pesaloop-dev
rm /tmp/pesaloop-secrets.yaml
```

## Deploying

| Action | What to do |
|--------|-----------|
| Deploy to dev | Push to main in PesaLoop repo — GitHub Actions handles it |
| Deploy to staging | Push a version tag: `git tag v1.0.0 && git push --tags` |
| Deploy to production | Open PR updating production/kustomization.yaml → merge → Sync in ArgoCD |
| Rollback | `git revert HEAD && git push` — ArgoCD applies automatically |

## Secrets

See `pesaloop/base/backend/secrets.yaml` for the template and instructions.
Use Sealed Secrets in production so encrypted secrets can live in Git safely.
