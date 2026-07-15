<!-- AUTO-SYNCED from agents KB: concepts/CLUSTER.md @ 4fe753e.
     Do NOT edit here — edit the source in ~/projects/agents and re-run scripts/sync-conventions.sh. -->

# Cluster

A single **k3s** cluster (`k3s-01`) hosts almost everything. It is managed purely by
GitOps — see [DEPLOYMENT.md](DEPLOYMENT.md) and [CICD.md](CICD.md). This file is the
map of what runs where and the conventions that apply cluster-wide.

## Topology

- **1 node**, `k3s-01` — control-plane + worker, k3s `v1.33.4`, Debian 13, amd64,
  public IP `195.201.171.111` (a Hetzner-class box). Everything schedules here.
- Sizing: ~16 vCPU, ~128 GiB RAM. As of last check ~23% CPU / ~71% memory used, so
  **memory is the scarce resource** — be conscious of JVM heap sizing on Spring Boot
  backends (several sit at 400–900 MiB each).
- Local contexts: `kubectl` context `local` targets this cluster; `rancher-desktop`
  is a throwaway local context. Rancher (`cattle-*` namespaces) provides the UI at
  `rancher.jannekeipert.de`.

## Ingress, DNS & TLS

- **Traefik** (k3s built-in) is the only ingress controller; every `Ingress` uses
  `ingressClassName: traefik`. The LB service is `svclb-traefik` in `kube-system`.
- **cert-manager** issues Let's Encrypt certs; HTTPS is standard for public hosts.
- Two public domains: `*.jannekeipert.de` (personal projects) and
  `*.cosy-hosting.net` (the Cosy product). Each app gets a subdomain; the host →
  namespace mapping is visible via `kubectl get ingress -A`.

## Namespace conventions

- **One namespace per project**, named after the project (`strata`, `covered`,
  `medals`, `blockworks`, `architecture-studio`, `dust-of-apollon`, …).
- Prod/staging splits get a namespace each: **Cosy Domain Provider** →
  `cosy-prod` + `cosy-staging`. (Do **not** confuse these with `cosy`, which is the
  separate Cosy game-server platform.)
- Some namespaces co-host related apps: `syncup` also runs the Robert Space Tracker
  backend; `drei-alben` + `bobs-archiv` together make up Bobs Archiv.
- Namespaces are auto-created by ArgoCD (`CreateNamespace=true`), never by hand.

## Platform / infrastructure namespaces

| Namespace(s)                     | What runs there |
|----------------------------------|-----------------|
| `argocd`                         | ArgoCD — the GitOps engine (`argo.jannekeipert.de`) |
| `kube-system`                    | k3s core, Traefik, sealed-secrets controller, reflector |
| `cert-manager`                   | TLS certificate issuance |
| `default`, `monitoring`          | kube-prometheus-stack (Prometheus, Alertmanager, Grafana), blackbox-exporter, pushgateway |
| `observability`                  | SigNoz (OTel traces/metrics/logs, ClickHouse) — `signoz.jannekeipert.de` |
| `minio`                          | S3-compatible object storage (`minio.jannekeipert.de`) |
| `velero`                         | cluster backups |
| `n8n`                            | workflow automation |
| `cattle-*`, `fleet-*`, `p-*`, `u-*` | Rancher / Fleet management plumbing — leave alone |

## Cluster-wide add-ons (managed in `cluster-deployment/infrastructure/`)

- **Sealed Secrets** — the only way secrets enter Git; seal with `kubeseal`.
- **Reflector / Reloader** — mirror secrets across namespaces; restart pods on
  ConfigMap/Secret change.
- **SigNoz** (`observability` namespace) is the standard observability platform for
  metrics, logs, and traces — see [MONITORING.md](MONITORING.md). The legacy
  **kube-prometheus-stack + Loki** stack still runs in parallel during migration but
  is being decommissioned. Spring backends expose `/actuator/prometheus` (Micrometer);
  the Cosy platform runs its own InfluxDB + Loki inside the `cosy` namespace.

## Working with the cluster (agent guidance)

- **Read-only is safe**: `kubectl get/describe/logs/top` for diagnosing.
- **Never mutate live state** to "fix" something (`apply`/`edit`/`scale`/`delete`
  a pod for config) — it will be reverted by ArgoCD `selfHeal`. Change the deployment
  repo instead.
- Deleting a pod to force a restart is acceptable for debugging (the Deployment
  recreates it), but prefer Reloader-driven restarts.
- To find where an app lives: `kubectl get ingress -A | grep <host>` →  namespace →
  `kubectl get pods -n <ns>`.
