# wharf-deployment — agent notes

Kustomize + ArgoCD manifests for the Wharf web layer. Single `main` environment,
namespace `wharf`, host `wharf.jannekeipert.de`. See [README.md](README.md).

## Conventions (auto-synced from the KB — do not edit the copies)

The docs under `docs/conventions/` are committed copies of the shared knowledge base
(`~/projects/agents`), kept fresh by `scripts/sync-conventions.sh`. Edit conventions at
the KB source, never here.

@docs/conventions/AGENT.md
@docs/conventions/wharf.md
@docs/conventions/DEPLOYMENT.md
@docs/conventions/CICD.md
@docs/conventions/CLUSTER.md

## Repo-specific rules

- **Never `kubectl apply` these manifests** — commit and let ArgoCD sync (`selfHeal`
  reverts manual changes). Read-only `kubectl get/describe/logs` is fine.
- The `images:` tags in `overlays/main/kustomization.yaml` are **CI-owned** — the
  product repos' `docker.yml` rewrites them via `kustomize edit set image`. Only
  hand-edit to bootstrap.
- Secrets are **SealedSecrets** only — plaintext never enters git. See
  [docs/secrets.md](docs/secrets.md).
