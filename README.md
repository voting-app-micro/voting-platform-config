# voting-platform-config

GitOps config repo for the voting platform — **Kustomize manifests** that define the desired state synced to AKS by **Argo CD**. This repo holds Kubernetes manifests only; no application code and no infrastructure/Terraform.

- App code + CI pipelines → `voting-platform-app`
- Infrastructure (Terraform) → `voting-platform-infra`
- **Deployment manifests (this repo)** → watched by Argo CD

---

## Repository structure

```
voting-platform-config/
├── base/                     # environment-agnostic manifests
│   ├── vote/                 # deployment.yaml, service.yaml
│   ├── result/               # deployment.yaml, service.yaml
│   ├── worker/               # deployment.yaml (no service — background worker)
│   ├── redis/                # deployment.yaml, service.yaml
│   ├── db/                   # deployment.yaml, service.yaml (Postgres)
│   └── kustomization.yaml    # lists all base resources
└── overlays/
    └── dev/
        └── kustomization.yaml  # namespace + per-service image tags
```

## Application overview

Five components (the standard example voting app):

| Component | Image | Service | Notes |
|-----------|-------|---------|-------|
| vote | `votingappregistry7.azurecr.io/votingapp-vote` | NodePort 31000 | Python front-end |
| result | `votingappregistry7.azurecr.io/votingapp-result` | NodePort 31001 | Node.js results page |
| worker | `votingappregistry7.azurecr.io/votingapp-worker` | none | .NET background worker |
| redis | `redis:alpine` (public) | ClusterIP 6379 | vote queue |
| db | `postgres:15-alpine` (public) | ClusterIP 5432 | vote store |

> **Service names matter:** the apps connect to `db` and `redis` by hostname (hardcoded, no env override), so those Service names must not change.

## How it fits together (GitOps flow)

```
voting-platform-app (CI)          voting-platform-config (this repo)        AKS
─────────────────────────         ──────────────────────────────────       ─────────
commit → build image → push  ──►  pipeline bumps image tag in              Argo CD
to ACR                            overlays/dev/kustomization.yaml   ──►     syncs ──► pods
```

- CI builds/pushes images to ACR, then updates the image tag in this repo.
- Argo CD watches this repo and reconciles the cluster to match.
- No `kubectl apply` from CI — git is the single source of truth.

## Image registry & tags (where to change them)

`base/` uses **registry-agnostic** image names (`votingapp-vote`, `votingapp-result`, `votingapp-worker`).
The **registry** and **tag** for each app live in ONE place — the `images:` block of the overlay:
[`overlays/dev/kustomization.yaml`](overlays/dev/kustomization.yaml):

```yaml
images:
  - name: votingapp-vote                                   # logical name (matches base)
    newName: votingappregistry7.azurecr.io/votingapp-vote  # registry — change here
    newTag: "17"                                           # tag — change here
```

- **Change the registry** → edit `newName` (per app).
- **Change the image tag** → edit `newTag` (per app).
- **Never** edit `base/` for registry/tag — base stays generic so multiple environments can reuse it.

The CI pipeline promotes a build by rewriting this block:

```bash
kustomize edit set image \
  votingapp-vote=votingappregistry7.azurecr.io/votingapp-vote:<BuildId>
```

## Local usage

Render manifests without applying:

```bash
kubectl kustomize overlays/dev
# or: kustomize build overlays/dev
```

Apply manually (for testing, before Argo CD is wired up):

```bash
kubectl create namespace voting
kubectl apply -k overlays/dev
```

## Argo CD

An Argo CD Application should point at `overlays/dev` with:

- `path: overlays/dev`
- `destination.namespace: voting`
- `syncOptions: CreateNamespace=true` (the overlay places resources in `voting` but does not create the namespace itself)
- `syncPolicy.automated` with `prune` + `selfHeal`

## Roadmap

- [ ] Add `overlays/staging` and `overlays/prod`
- [ ] Argo CD `ApplicationSet` to manage all environments
- [ ] Manual sync gate for prod
- [ ] Admission policies (Kyverno) — reject unsigned images
