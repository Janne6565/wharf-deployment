# wharf-deployment — agent notes

Kustomize + ArgoCD manifests for the Wharf web layer. Single `main` environment,
namespace `wharf`, host `wharf.jannekeipert.de`. See [README.md](README.md).

## Repo-specific rules

- **Never `kubectl apply` these manifests** — commit and let ArgoCD sync (`selfHeal`
  reverts manual changes). Read-only `kubectl get/describe/logs` is fine.
- The `images:` tags in `overlays/main/kustomization.yaml` are **CI-owned** — the
  product repos' `docker.yml` rewrites them via `kustomize edit set image`. Only
  hand-edit to bootstrap.
- Secrets are **SealedSecrets** only — plaintext never enters git. See
  [docs/secrets.md](docs/secrets.md).
