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
| `wharf-oauth-secret` | `OAUTH_GOOGLE_CLIENT_ID`, `OAUTH_GOOGLE_CLIENT_SECRET`, `OAUTH_GITHUB_CLIENT_ID`, `OAUTH_GITHUB_CLIENT_SECRET` | backend Google/GitHub social login. **Optional** — referenced with `optional: true`; while absent, `GET /auth/oauth/providers` returns `[]` and the web app keeps the buttons disabled. Any subset of providers works: configure only the pair(s) you have |
| `wharf-mail-secret` | `MAIL_SERVICE_API_KEY` | backend project-invite notification emails via the mail-service (`POST https://mail-service.jannekeipert.de/api/v1/send`, `Authorization: Bearer <key>`). The key is a send-scoped mail-service API key (label `wharf-backend`) bound to the `info@jannekeipert.de` SMTP connection — mint/revoke in the Mail Manager UI or API. **Optional** — when absent the backend uses its no-op mail client and invites simply send no email |

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

```sh
# OAuth client credentials (once obtained from the provider consoles — see below).
kubectl create secret generic wharf-oauth-secret -n wharf \
  --dry-run=client -o yaml \
  --from-literal=OAUTH_GOOGLE_CLIENT_ID="..." \
  --from-literal=OAUTH_GOOGLE_CLIENT_SECRET="..." \
  --from-literal=OAUTH_GITHUB_CLIENT_ID="..." \
  --from-literal=OAUTH_GITHUB_CLIENT_SECRET="..." \
  | kubeseal --cert pub-cert.pem --format yaml \
  > overlays/main/sealed-secrets/wharf-oauth-secret.yaml
```

```sh
# Mail-service API key (mint a send-scoped key for the info@jannekeipert.de
# connection via the Mail Manager, then seal it here).
kubectl create secret generic wharf-mail-secret -n wharf \
  --dry-run=client -o yaml \
  --from-literal=MAIL_SERVICE_API_KEY="mk_..." \
  | kubeseal --cert pub-cert.pem --format yaml \
  > overlays/main/sealed-secrets/wharf-mail-secret.yaml
```

The backend Deployment carries `reloader.stakater.com/auto: "true"`, so a synced
secret change triggers a rolling restart automatically.

## OAuth app registration (provider consoles)

Register one OAuth app per provider and use these settings:

- **Google** ([console.cloud.google.com](https://console.cloud.google.com/apis/credentials),
  "OAuth client ID", type *Web application*):
  - Authorized redirect URI: `https://wharf.jannekeipert.de/api/v1/auth/oauth/google/callback`
  - (dev: `http://localhost:8080/api/v1/auth/oauth/google/callback`)
- **GitHub** ([github.com/settings/developers](https://github.com/settings/developers),
  "New OAuth App"):
  - Authorization callback URL: `https://wharf.jannekeipert.de/api/v1/auth/oauth/github/callback`
  - (GitHub allows one callback per app — create a second app for local dev.)

The public base URL used to build these redirect URIs is `OAUTH_PUBLIC_BASE_URL` in
`base/backend/configmap.yaml`.

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
