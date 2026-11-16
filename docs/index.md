# python-backend-gitops

Deployment (CD) of [python-backend](https://github.com/Everesto-Inc/python-backend), DevOps space **orchid**, managed by Backstage.
Backstage has **no cluster access**: it writes git and Infisical; an admin runs the commands below on the control plane
(`ssho control-plane`). Every command is safe to run again.

## What runs where

```mermaid
flowchart TB
  subgraph git[GitHub python-backend-gitops]
    D[apps/dev]
    P[apps/prod]
    A[argocd/*.yaml]
  end
  subgraph inf[Infisical project python-backend-gitops]
    ID[(env dev)]
    IP[(env prod)]
  end
  subgraph argo[ns argocd]
    AD[Application dev-python-backend]
    AP[Application prod-python-backend]
  end
  subgraph es[ns external-secrets]
    L[Secret infisical-python-backend-auth]
  end
  SD{{ClusterSecretStore dev-python-backend}}
  SP{{ClusterSecretStore prod-python-backend}}
  subgraph nd[ns dev-python]
    PD[pod python-backend]
  end
  subgraph np[ns prod-python]
    PP[pod python-backend]
  end
  A -. kubectl apply .-> AD & AP & SD & SP
  D -->|Sync| AD --> nd
  P -->|Sync| AP --> np
  L --> SD & SP
  ID --> SD -->|ESO: Secret python-backend| nd
  IP --> SP -->|ESO: Secret python-backend| np
```

| Object | Kind | Namespace | Created by |
|---|---|---|---|
| `infisical-python-backend-auth` | Secret (Infisical login) | external-secrets | you — step 2 |
| `infisical-python-backend-dev` / `-prod` | ClusterSecretStore | — (cluster-wide) | you — steps 3 / 7 |
| `orchid` | AppProject | argocd | you — step 4 |
| `dev-python-backend` / `prod-python-backend` | Application | argocd | you — steps 4 / 7 |
| `dev-python` / `prod-python` | Namespace | — | ArgoCD on Sync |
| `python-backend` | Deployment, Service, ExternalSecret | dev-python / prod-python | ArgoCD on Sync |
| `python-backend` | Secret (the app's secrets) | dev-python / prod-python | ESO, from Infisical |

Why the files have no `namespace:` line: it is set once per folder in `apps/<env>/kustomization.yaml`
(`namespace: dev-python` / `prod-python`) and in each Application's `destination`.

## Once (per cluster / per app)

### 1. Tenant node label (once per cluster)

```bash
sudo kubectl label node dev-k3s-argocd everesto.live/pool=tenants --overwrite
```

Every tenant Deployment has nodeSelector everesto.live/pool=tenants. Without this label the pods stay Pending. Already done if any other app runs.

**Creates:**

- label everesto.live/pool=tenants (node dev-k3s-argocd)

### 2. Infisical login for ESO (identity eso-python-backend)

```bash
sudo kubectl -n external-secrets create secret generic infisical-python-backend-auth \
  --from-literal=clientId='<CLIENT_ID>' --from-literal=clientSecret='<CLIENT_SECRET>'
```

The login ESO uses to READ the Infisical project python-backend-gitops (dev and prod). Real values are never stored: Backstage → python-backend-gitops → Secrets → "New ESO login" shows this command with them, once. Lives in external-secrets, which tenants cannot read.

**Creates:**

- Secret infisical-python-backend-auth (ns external-secrets)

## Dev

### 3. Secret store for dev

```bash
sudo kubectl apply -f https://raw.githubusercontent.com/Everesto-Inc/python-backend-gitops/main/argocd/secret-stores.yaml
```

Tells ESO where dev secrets come from: Infisical project python-backend-gitops, environment dev, path /, using the login of step 2. Its condition allows ONLY namespace dev-python to use it.

**Creates:**

- ClusterSecretStore infisical-python-backend-dev (cluster-wide)

### 4. ArgoCD project orchid + application dev-python-backend

```bash
sudo kubectl apply -f https://raw.githubusercontent.com/Everesto-Inc/python-backend-gitops/main/argocd/argocd.yaml
```

AppProject orchid = guard rails: only *-gitops repos, only namespaces dev-python / prod-python. Application dev-python-backend = an instruction to ArgoCD: "take folder apps/dev of python-backend-gitops and put it into dev-python, sync manually". It creates no pods yet — it appears as a tile in ArgoCD.

**Creates:**

- AppProject orchid (ns argocd)
- Application dev-python-backend (ns argocd)

### 5. Sync dev-python-backend (or ArgoCD UI → Sync)

```bash
sudo kubectl -n argocd patch application dev-python-backend --type merge -p '{"operation":{"sync":{}}}'
```

ArgoCD applies everything in apps/dev: the namespace first, then Deployment (image + tag, port, /health probes, memory), Service (port 80 → container port) and ExternalSecret. ESO sees the ExternalSecret and creates Secret python-backend from Infisical dev; the pod gets every key as an environment variable. Run again after every "Deploy to dev".

**Creates:**

- Namespace dev-python
- Deployment + pod python-backend (ns dev-python)
- Service python-backend (ns dev-python)
- ExternalSecret python-backend (ns dev-python)
- Secret python-backend (ns dev-python, created by ESO)
- RoleBinding backstage-read-python-backend (ns dev-python: Backstage may READ this namespace — CD tab)

### 6. Check dev

```bash
sudo kubectl -n argocd get application dev-python-backend
sudo kubectl -n dev-python get pods,externalsecret,secret
```

Application Synced/Healthy, pod Running, ExternalSecret SecretSynced, Secret present.

## Prod

### 7. Prod secret store + application prod-python-backend — only after the first "Promote to prod" PR is merged (not done yet)

```bash
sudo kubectl apply -f https://raw.githubusercontent.com/Everesto-Inc/python-backend-gitops/main/argocd/argocd-prod.yaml
```

Same as steps 3 + 4 for prod: the store reads Infisical environment prod and only namespace prod-python may use it; the Application takes apps/prod into prod-python. The file exists only after the first promote PR is merged (404 before). Set the real prod secrets in the Secrets tab before syncing.

**Creates:**

- ClusterSecretStore infisical-python-backend-prod (cluster-wide)
- Application prod-python-backend (ns argocd)

### 8. Sync prod-python-backend (or ArgoCD UI → Sync)

```bash
sudo kubectl -n argocd patch application prod-python-backend --type merge -p '{"operation":{"sync":{}}}'
```

Like step 5, for prod. Run again after every merged "Promote to prod" PR.

**Creates:**

- Namespace prod-python
- Deployment + pod python-backend (ns prod-python)
- Service python-backend (ns prod-python)
- ExternalSecret python-backend (ns prod-python)
- Secret python-backend (ns prod-python, created by ESO)
- RoleBinding backstage-read-python-backend (ns prod-python: Backstage may READ this namespace — CD tab)

### 9. Check prod

```bash
sudo kubectl -n argocd get application prod-python-backend
sudo kubectl -n prod-python get pods,externalsecret,secret
```

Application Synced/Healthy, pod Running, ExternalSecret SecretSynced, Secret present.

## Day to day

| I want to… | Do |
|---|---|
| deploy a new build to dev | Backstage → CD card → **Deploy to dev** → pick a build → step 5 (Sync) |
| put dev's version into prod | Backstage → **Promote to prod** → merge the PR → step 8 (Sync) |
| roll back | **Deploy to dev** with an older build (the last 5 images are kept) → Sync; then promote |
| change a secret | Backstage → python-backend-gitops → **Secrets** → dev/prod (prod: maintainers/owners) → then: |

```bash
# apply a changed secret now (dev; use prod-python for prod) — only when the app already runs there
sudo kubectl -n dev-python annotate externalsecret python-backend force-sync=$(date +%s) --overwrite
sudo kubectl -n dev-python rollout restart deployment python-backend
```

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `namespaces "prod-python" not found` | prod not synced yet | steps 7 + 8 |
| `argocd-prod.yaml` → 404 | first promote PR not merged yet (or GitHub cache, ~5 min) | merge the PR, wait |
| pod `Pending` | node label missing | step 1 |
| ExternalSecret `SecretSyncedError` | login secret missing/wrong, or store missing | step 2 ("New ESO login"), steps 3 / 7 |
| pod `ImagePullBackOff` | image package is private | GitHub → Packages → python-backend → make public |
| pod `CrashLoopBackOff` | app fails at start — often a `CHANGE_ME` secret | Secrets tab → set the value → apply |
| Application `OutOfSync` | expected: sync is manual | Sync (steps 5 / 8) |

