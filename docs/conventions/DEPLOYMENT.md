<!-- AUTO-SYNCED from agents KB: concepts/DEPLOYMENT.md @ 4fe753e.
     Do NOT edit here — edit the source in ~/projects/agents and re-run scripts/sync-conventions.sh. -->

# Deployment

Every deployable project follows the same **GitOps** model: a dedicated
`*-deployment` repo holds Kubernetes manifests, ArgoCD watches it, and CI bumps the
image tag. Nothing is ever `kubectl apply`-ed by hand in a running environment.

## The pieces

- **Product repos** (`*-backend`, `*-frontend`) — application code + `Dockerfile`.
  CI builds and pushes an image to **ghcr.io**.
- **Deployment repo** (`*-deployment`) — Kustomize manifests only. This is the
  single source of truth for what runs in the cluster.
- **ArgoCD** — an `Application` points at a path in the deployment repo and
  continuously reconciles the cluster to match Git.

## Kustomize layout

Standard `base` + `overlays` structure (see `strata-deployment`, `cosy-domain-provider-deployment`):

```
<project>-deployment/
├── base/                 # deployment.yaml, service.yaml, configmap.yaml per component
│   ├── backend/          #   (+ rbac.yaml / serviceaccount.yaml when the app needs cluster access)
│   ├── frontend/
│   ├── postgres/         # statefulset for stateful stores
│   └── kustomization.yaml
├── overlays/
│   └── main/             # or staging/ + prod/ for projects with an environment split
│       ├── kustomization.yaml   # namespace + ingress + `images:` tag pins
│       └── ingress.yaml
└── argocd/
    └── main-app.yaml     # the ArgoCD Application (only for standalone apps)
```

**DO:**
- Keep all environment-invariant manifests in `base/`; put namespace, ingress, and
  image-tag pins in the overlay.
- Pin images in the overlay's `kustomization.yaml` under `images:` — CI rewrites the
  `newTag` via `kustomize edit set image` (see [CICD.md](CICD.md)). Treat the tag
  value as CI-owned; don't hand-edit it except to bootstrap.
- Use a StatefulSet + PVC for Postgres/InfluxDB/MinIO; a Deployment for stateless
  backends and frontends.
- Give backends that talk to the Kubernetes API (e.g. Strata) their own
  `ServiceAccount` + RBAC `Role/RoleBinding` in `base/backend/`.

**DON'T:**
- `kubectl apply`, `kubectl edit`, or `helm upgrade` directly against a namespace —
  change Git and let ArgoCD sync. Manual changes get reverted by `selfHeal`.
- Bake secrets into manifests — use **Sealed Secrets** (`kubeseal`); the controller
  runs in `kube-system`. Plaintext secrets never enter Git.

## ArgoCD Application conventions

```yaml
spec:
  source: { repoURL: https://github.com/janne6565/<project>-deployment.git,
            targetRevision: main, path: overlays/main }
  destination: { server: https://kubernetes.default.svc, namespace: <project> }
  syncPolicy:
    automated: { prune: true, selfHeal: true }
    syncOptions: [ CreateNamespace=true ]
```

- **Fully automated**: `prune: true` + `selfHeal: true` — any commit to `main`
  (a CI tag bump or a manual manifest edit) reconciles with no manual step.
- One namespace per project (`CreateNamespace=true`). Projects with a prod/staging
  split get one namespace each (e.g. `cosy-prod`, `cosy-staging`).
- Canonical repo-URL casing, pinned default branch, and a finalizer on every app.

## `cluster-deployment` — the single ArgoCD entry point

**`cluster-deployment`** (`github.com/Janne6565/cluster-deployment`, clone to
`~/projects/cluster-deployment`) is the one repo that defines the entire `k3s-01`
cluster as an **app-of-apps**. There is exactly one manually-created Application —
`root` — and it manages every other Application, AppProject, and adopted Helm release
in the cluster, **including itself**. To change anything cluster-wide, you commit here.

### `root.yaml` — the root Application

```yaml
# Application "root" in namespace argocd, project "cluster"
source: { repoURL: .../cluster-deployment, targetRevision: main, path: ., directory: { recurse: true } }
syncPolicy: { automated: { prune: true, selfHeal: true } }
# NOTE: root has NO finalizer — a mistaken deletion detaches, it does not cascade-delete.
```

It recurses the whole repo, so any manifest committed anywhere in it becomes managed
automatically.

### Layout

```
cluster-deployment/
├── root.yaml          # the one root Application (self-managing)
├── projects/          # one ArgoCD AppProject per project (+ `cluster`, `default`);
│                      #   sync-wave -1 so projects exist before the apps that use them
├── apps/              # top-level Applications:
│                      #   • standalone apps (architecture-studio, blockworks, strata, portfolio…)
│                      #   • app-of-apps PARENTS (cosy-deployment, cosy-domain-provider-deployment,
│                      #     mail-service-deployment) whose child apps live in the product
│                      #     deployment repos — e.g. cosy-deployment → Magenta-Mause/Cosy-Internal-Deployment
├── infrastructure/    # adopted Helm releases as Applications: cert-manager, kube-prometheus-stack,
│                      #   loki, minio, sealed-secrets, reflector, reloader, traefik-helmchartconfig,
│                      #   signoz (+ signoz-k8s-infra), blackbox-exporter, uptime-*
└── BOOTSTRAP.md       # how to stand the cluster up from nothing
```

### Adding / changing something

- **New standalone app** → add an `Application` to `apps/` (and an `AppProject` to
  `projects/` if it needs its own), commit. `root` syncs it in — no `kubectl`.
- **New product with its own deployment repo** → add an app-of-apps **parent** to
  `apps/` pointing at that repo's argo path (like `cosy-deployment`); its children are
  defined over there.
- **New infra Helm release** → add an `Application` to `infrastructure/`.
- Per-Application conventions (from the repo README): canonical repo-URL casing, pinned
  default branch, `prune: true` + `selfHeal: true`, a finalizer on every app **except
  `root`**, and the correct `AppProject`.

### Bootstrap (chicken-and-egg)

Everything is GitOps except a few objects applied by hand **once**, in order (see
`BOOTSTRAP.md`): (1) the git-credentials secret `github-creds-janne` in `argocd` — a
`repo-creds` template for the `https://github.com/Janne6565` prefix so ArgoCD can read
the **few** private repos (most Janne6565 repos are public; Magenta-Mause repos are all
public), (2) the MinIO
root-credentials secret (kept off-git so `prune` never touches it), (3) the `cluster`
AppProject (`kubectl apply -f projects/cluster.yaml` — `root` runs in it, so it must
exist first), then (4) `kubectl apply -f root.yaml`. After that `root` adopts and
self-manages all four.

Cluster topology and namespaces are documented in [CLUSTER.md](CLUSTER.md).
