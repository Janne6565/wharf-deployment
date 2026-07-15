# wharf-deployment

Kubernetes manifests for **Wharf** (the web layer: Spring Boot sync/auth backend +
TanStack Start web client + Postgres) on the `k3s-01` cluster, managed with
[Kustomize](https://kustomize.io/) and delivered by
[ArgoCD](https://argo-cd.readthedocs.io/) (GitOps).

Deployed to **https://wharf.jannekeipert.de** in the `wharf` namespace.

There is a **single environment** — `main`. Every push to a product repo's `main`
branch builds an image and bumps its tag here; ArgoCD reconciles the change onto the
cluster automatically. There is no staging/prod split and no manual promotion.

## Topology

```
                       ┌─────────────────────────────────────────┐
  wharf.jannekeipert.de│  Traefik Ingress (cert-manager TLS)      │
        ───────────────┤   /api  → backend:8080                   │
                       │   /     → web:3000                        │
                       └───────────┬─────────────────┬────────────┘
                                   │                 │
                        ┌──────────▼──────┐   ┌──────▼───────────┐
                        │ backend (Spring)│   │ web (TanStack    │
                        │ :8080, prod     │   │ Start / Nitro)   │
                        │ Flyway + JPA    │   │ :3000 SSR shell  │
                        └──────────┬──────┘   └──────────────────┘
                                   │
                        ┌──────────▼──────┐
                        │ postgres:17     │  StatefulSet + 8Gi PVC
                        │ db/user `wharf` │
                        └─────────────────┘
```

- Browser traffic is **same-origin**: the web client's Axios base URL is relative
  (`/api/v1/...`), so the ingress routes API calls to the backend with no CORS.
- The web auth routes are `ssr:false` (all crypto is client-side), so the Node
  server only ever returns the static SSR shell — it never calls the backend itself.

## Layout

```
base/
  backend/     Deployment + Service + ConfigMap (Spring Boot, prod profile)
  web/         Deployment + Service (TanStack Start Node server, :3000)
  postgres/    StatefulSet + headless Service (state DB, 8Gi PVC)
overlays/
  main/        the only environment: namespace `wharf`, Ingress, image pins,
               sealed-secrets/  (SealedSecrets — safe to commit)
argocd/
  main-app.yaml   reference copy of the ArgoCD Application (the authoritative
                  Application lives in cluster-deployment/apps/wharf.yaml)
docs/
  secrets.md      how the SealedSecrets workflow works
```

## How it deploys — CI owns the image tag

```
push to wharf-backend / wharf-web  (main)
  → CI (ci.yml) lints + tests + builds
  → CI (docker.yml) builds & pushes ghcr.io/janne6565/wharf-{backend,web}:main-<sha>
  → CI clones this repo, `kustomize edit set image` in overlays/main, commits + pushes
  → ArgoCD detects the commit and syncs the cluster
```

**Merge to main in a product repo IS the prod deploy.** The overlay pins the immutable
`main-<sha>` tag (never a moving `latest`) so every state is reproducible and a rollback
is a `git revert` here.

## Secrets

All secrets are committed as **SealedSecrets** (encrypted for this cluster's
`sealed-secrets-controller` in `kube-system`) — see [docs/secrets.md](docs/secrets.md).

| SealedSecret | Key | Used by |
| --- | --- | --- |
| `wharf-db-secret` | `POSTGRES_PASSWORD` | postgres (`POSTGRES_PASSWORD`) **and** backend (`DB_PASSWORD`) — one shared value |
| `wharf-jwt-secret` | `JWT_SECRET_KEY` | backend JWT signing (differs from the committed dev fallback) |

ghcr.io packages `wharf-backend` / `wharf-web` are **public**, so no image-pull secret
is needed.

## Bootstrap (new cluster)

1. Ensure the `wharf` AppProject and `wharf` Application exist in `cluster-deployment`
   (`projects/wharf.yaml`, `apps/wharf.yaml`) — the root app syncs them in.
2. DNS: `wharf.jannekeipert.de` → `195.201.171.111` (the cluster ingress IP).
3. ArgoCD creates the `wharf` namespace, the controller decrypts the SealedSecrets,
   Postgres initialises, the backend runs Flyway, cert-manager issues the TLS cert.

No manual `kubectl apply` is needed for normal deployments.

## Related repositories

| Repository | Description |
| --- | --- |
| `wharf-backend` | Java 21 + Spring Boot — sync + device-code auth; builds the backend image |
| `wharf-web`     | React 19 + TanStack Start — landing + web auth flow; builds the web image |
