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

> ⚠️ **Not automated yet (TODO — tomorrow):** the CI pipeline currently pushes to ACR but does
> **not** update this repo. The image tag in `overlays/dev/kustomization.yaml` is bumped **manually**
> for now. Closing this gap (pipeline runs `kustomize edit set image` + commits to this repo) is what
> makes the loop fully automatic. See [Roadmap](#roadmap).

## Prerequisites

- **Tools:** `az` CLI, `kubectl`, and `kustomize` (or `kubectl` ≥ 1.14 which has `-k` built in).
- **Azure:** access to the subscription holding the ACR (`votingappregistry7`) and AKS cluster.
- **A running AKS cluster** with **Argo CD installed** — see
  [`voting-platform-infra/Manualconfigurations.md`](../voting-platform-infra/Manualconfigurations.md).
- Images already built and pushed to ACR by the CI pipelines in `voting-platform-app`.

## Deploy (step-by-step)

> First-time end-to-end deployment. Steps 1–3 are infra (done once); 4–8 wire up and verify this app.

**1. Connect to the cluster & ensure ACR pull auth**
```bash
az aks get-credentials -n azuredevops -g voting-app-project --overwrite-existing
az aks update -n azuredevops -g voting-app-project --attach-acr votingappregistry7   # pull auth
kubectl get nodes                                                                    # all Ready?
```

**2. Confirm Argo CD is running** (install steps in the infra doc)
```bash
kubectl get pods -n argocd    # all Running/Ready
```

**3. Access the Argo CD UI**
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
# open https://localhost:8080  (user: admin; password from argocd-initial-admin-secret)
```

**4. Register this config repo in Argo CD**
- UI: **Settings → Repositories → + Connect Repo → VIA HTTPS**
  - URL: `https://github.com/voting-app-micro/voting-platform-config.git`
  - Username + **GitHub PAT** (read scope) if the repo is private (public repos need no auth)
- Status should show **Successful**.

**5. Create the Argo CD Application** (points at `overlays/dev`)

Save as `application.yaml` and apply, or create the same via the UI:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: voting-dev
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/voting-app-micro/voting-platform-config.git
    targetRevision: main
    path: overlays/dev
  destination:
    server: https://kubernetes.default.svc
    namespace: voting
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true   # auto-creates the "voting" namespace
```
```bash
kubectl apply -n argocd -f application.yaml
```

**6. Sync** — with `automated` above it self-syncs; otherwise click **Sync** in the UI.

**7. Verify it worked**
```bash
kubectl get pods -n voting        # all 5 (vote/result/worker/redis/db) Running
kubectl get svc  -n voting        # vote + result services present
# In the Argo CD UI the app should show:  Synced  +  Healthy
```

**8. Access the app UI** (port-forward — see [Accessing the app](#accessing-the-app))

---

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

## Accessing the app (vote & result UIs)

Only `vote` and `result` have web UIs (`worker`/`db`/`redis` are backend). Nodes have no public IP by
default, so the `NodePort` (31000/31001) in the manifests isn't reachable from your laptop — use
**port-forward** (simplest) or switch the services to a **LoadBalancer**.

**1. Confirm the services are up first**
```bash
kubectl get pods -n voting        # vote + result Running 1/1
kubectl get svc  -n voting        # vote (8080) and result (8081) present
```

**2. Port-forward each UI** — run each in its own terminal and leave it open:
```bash
# Vote UI (cast a vote)     → http://localhost:5000
kubectl port-forward -n voting svc/vote   5000:8080

# Result UI (live tally)    → http://localhost:5001
kubectl port-forward -n voting svc/result 5001:8081
```

> `localPort:servicePort` — the `vote` service listens on `8080`, `result` on `8081`. Pick any free
> local port (e.g. `8080:8080`) if `5000`/`5001` are already in use.

**3. Open the UIs and test the flow**
- Open **http://localhost:5000**, cast a vote (Cats vs Dogs).
- Open **http://localhost:5001** — the tally updates live. Seeing the vote land there proves the whole
  path works end-to-end: `vote → redis → worker → postgres → result`.

Quick smoke test without a browser:
```bash
curl -sSf http://localhost:5000 >/dev/null && echo "vote UI OK"
curl -sSf http://localhost:5001 >/dev/null && echo "result UI OK"
```

> A port-forward dies when you close the terminal or the pod restarts — just re-run it. The commands
> are identical on Windows (PowerShell) and macOS/Linux.

### Public URL (optional — costs an Azure public IP, no TLS, demo only)
```bash
kubectl patch svc vote   -n voting -p '{"spec":{"type":"LoadBalancer"}}'
kubectl patch svc result -n voting -p '{"spec":{"type":"LoadBalancer"}}'
kubectl get svc -n voting -w    # wait for EXTERNAL-IP to populate
# then browse http://<EXTERNAL-IP>:8080  and  http://<EXTERNAL-IP>:8081
```

To revert to internal-only: `kubectl patch svc vote -n voting -p '{"spec":{"type":"NodePort"}}'`
(same for `result`).

## Image pull authentication (ACR ↔ AKS)

**Approach: kubelet managed identity + `AcrPull` (ACR-attach). No static secrets.**

The AKS node's managed identity is granted the `AcrPull` role on the registry, so the kubelet
pulls images using auto-rotating Azure tokens — no `imagePullSecrets`, no passwords stored in the
cluster. This is the production-correct mechanism; the only prod hardening on top is to define the
role assignment in Terraform (Phase 2) and optionally put ACR behind a private endpoint.

> We deliberately avoid `imagePullSecrets` with a service-principal username/password — that's a
> legacy anti-pattern (secrets to rotate/leak). Managed identity is the standard.

### Steps to enable (one-time)

```bash
# 1. Attach the ACR to the AKS cluster (grants the kubelet identity AcrPull)
az aks update -n azuredevops -g voting-app-project --attach-acr votingappregistry7

# 2. (Optional) verify the AcrPull role assignment exists
az role assignment list \
  --scope $(az acr show -n votingappregistry7 --query id -o tsv) \
  --query "[?roleDefinitionName=='AcrPull'].principalName" -o table

# 3. Restart the stuck pods so they retry the pull with the new permission
kubectl delete pod -n voting --all

# 4. Confirm the app pods are now pulling and running
kubectl get pods -n voting
```

After step 3, `ImagePullBackOff` / `ErrImagePull` on the ACR images (vote/result/worker) should
clear. Public images (redis, postgres) are unaffected — they never needed ACR auth.

> **Phase 2:** move this into IaC — assign `AcrPull` to the kubelet identity via Terraform
> (`azurerm_role_assignment`) instead of the manual `az aks update`.

## Troubleshooting

### `namespaces "voting" not found` (SyncFailed / Missing)
The overlay places resources in the `voting` namespace but does not create it. Fix by enabling
`CreateNamespace=true` on the Argo CD Application (App Details → Sync Options → Auto-Create Namespace),
or create it once manually: `kubectl create namespace voting`.

### `ImagePullBackOff` / `ErrImagePull` on vote / result / worker
The node can't pull the app images from ACR. Public images (redis, postgres) pulling fine while only
the ACR images fail points to one of two causes — check the pod events first:

```bash
kubectl describe pod -n voting <pod-name> | tail -20   # read the Events section
```

| Event message | Cause | Fix |
|---------------|-------|-----|
| `unauthorized` / `authentication required` | AKS has no pull rights on the registry | Attach ACR to AKS: `az aks update -n <aks> -g <rg> --attach-acr votingappregistry7`, then `kubectl delete pod -n voting --all` to retry |
| `manifest for ...:<tag> not found` | The tag in the overlay doesn't exist in ACR | Verify with `az acr repository show-tags -n votingappregistry7 --repository votingapp-<svc> -o table` and correct `newTag` in `overlays/dev/kustomization.yaml` |
| `no such host` | Wrong registry name in `newName` | Fix the registry in the overlay's `images:` block |

> The registry/tag for each app is controlled **only** in `overlays/dev/kustomization.yaml` (`images:` block).
> `base/` is registry-agnostic — never put a registry or tag there.

### Argo CD components crash-looping (`repo-server` / `application-controller` not Ready)
Symptom: `kubectl get pods -n argocd` shows `argocd-repo-server` and/or `argocd-application-controller`
with many restarts or stuck `0/1`, and their events show `Readiness/Liveness probe failed: ... context
deadline exceeded`. The container's `lastState` is `Completed` (exit 0) — i.e. killed by the failed
**liveness** probe, so this is **not** OOM and **not** a config error.

```bash
kubectl get pods -n argocd
kubectl describe pod -n argocd <argocd-repo-server-pod> | tail -20   # look for "context deadline exceeded"
```

Cause: **node resource pressure.** Argo CD pods ship as `BestEffort` (no CPU/memory requests), so on a
saturated node they get CPU-throttled and can't answer their health probes in time — the kubelet then
restarts them in a loop. On a single 2-vCPU node (`Standard_D2lds_v6`, ~1.9 vCPU / 2.8 GB allocatable)
the voting app plus the full Argo CD control plane push CPU/memory **requests to ~90%+**, leaving no
headroom. (Slow image pulls — tens of seconds — are another symptom of the same starvation.)

```bash
kubectl describe node <node> | grep -A2 "Allocated resources"   # % of CPU/mem requested
kubectl top nodes                                               # live usage (needs metrics-server)
```

Fix — give the node headroom (add a node, or move to a bigger SKU), then bounce the stuck components:
```bash
az aks nodepool scale -g voting-app-project --cluster-name azuredevops -n agentpool --node-count 2

kubectl rollout restart deploy/argocd-repo-server deploy/argocd-server -n argocd
kubectl rollout restart statefulset/argocd-application-controller -n argocd
```

> Optionally set explicit resource `requests` on the Argo CD components so the scheduler stops treating
> them as disposable — but on a saturated node that alone won't help; add capacity first.

## Roadmap

- [ ] **(TOMORROW) Automate CI→CD promotion** — make the `voting-platform-app` pipeline update this
      repo after pushing to ACR: run `kustomize edit set image votingapp-<svc>=<registry>/votingapp-<svc>:<BuildId>`
      in `overlays/dev`, commit, and push. This closes the manual gap so a code commit flows all the
      way to the cluster without hand-editing `overlays/dev/kustomization.yaml`. *(Prefer a PR-based
      update + digest pinning over a direct push to `main`.)*
- [ ] Add `overlays/staging` and `overlays/prod`
- [ ] Argo CD `ApplicationSet` to manage all environments
- [ ] Manual sync gate for prod
- [ ] Admission policies (Kyverno) — reject unsigned images
