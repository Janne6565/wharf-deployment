# Secrets — how sealed secrets work here

All secrets in this repo are **SealedSecrets**: asymmetrically encrypted with the
public key of the `sealed-secrets-controller` running in `kube-system`. Only that
controller can decrypt them, so the encrypted YAML is safe to commit — ArgoCD applies
the `SealedSecret`, the controller decrypts it and materialises a normal `Secret` with
the same name in the same namespace.

A sealed secret is bound to its **name and namespace** (default "strict" scope):
re-sealing for a different name/namespace requires re-encrypting the value.

## The secrets

| SealedSecret | Keys | Used by |
| --- | --- | --- |
| `wharf-db-secret` | `POSTGRES_PASSWORD` | postgres StatefulSet (`POSTGRES_PASSWORD`) and the backend (`DB_PASSWORD`, via `secretKeyRef` on the same key) — a single shared password |
| `wharf-jwt-secret` | `JWT_SECRET_KEY` | backend JWT access/refresh signing. Must differ from the committed dev fallback (`application.properties`), or the app fail-closes in prod |

## Sealing / rotating a secret

```sh
# One-time: fetch the cluster's public cert (safe to keep around).
kubeseal --controller-namespace kube-system --fetch-cert > pub-cert.pem

# Build the plain Secret locally (NEVER applied), pipe through kubeseal, commit the
# sealed output. The plaintext must never touch git or survive on disk.
DB_PW=$(openssl rand -base64 24)
kubectl create secret generic wharf-db-secret -n wharf \
  --dry-run=client -o yaml \
  --from-literal=POSTGRES_PASSWORD="$DB_PW" \
  | kubeseal --cert pub-cert.pem --format yaml \
  > overlays/main/sealed-secrets/wharf-db-secret.yaml

JWT=$(openssl rand -base64 48)
kubectl create secret generic wharf-jwt-secret -n wharf \
  --dry-run=client -o yaml \
  --from-literal=JWT_SECRET_KEY="$JWT" \
  | kubeseal --cert pub-cert.pem --format yaml \
  > overlays/main/sealed-secrets/wharf-jwt-secret.yaml
```

The backend Deployment carries `reloader.stakater.com/auto: "true"`, so a synced
secret change triggers a rolling restart automatically.

## Caveats

- **Rotating the Postgres password** in `wharf-db-secret` does **not** change it in the
  running database — Postgres only reads `POSTGRES_PASSWORD` on first init. Run
  `ALTER USER wharf WITH PASSWORD '...'` in the DB as well (or wipe the PVC on a fresh
  deploy).
- **Rotating `JWT_SECRET_KEY`** invalidates every issued access/refresh token.
- **Back up the controller's private key** — without it nothing here can be decrypted
  again after a cluster rebuild:
  `kubectl get secret -n kube-system -l sealedsecrets.bitnami.com/sealed-secrets-key -o yaml > master-key.yaml`
  (store it somewhere safe, NOT in git).
