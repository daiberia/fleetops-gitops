# fleetops-gitops

GitOps configuration repository for the [FleetOps](https://github.com/daiberia/fleetops) platform. ArgoCD polls this repo and applies any change to the AKS cluster automatically.

**Do not apply manifests manually.** Push to this repo and ArgoCD handles the rest.

---

## How it works

The CI pipeline in `daiberia/fleetops` builds a container image, pushes it to ACR, and updates the `tag` field in the relevant `values.yaml` here. ArgoCD detects the commit and triggers a rolling update on the cluster.

```
daiberia/fleetops  →  CI updates values.yaml  →  daiberia/fleetops-gitops
                                                          │
                                                    ArgoCD polls
                                                          │
                                                    helm template
                                                          │
                                                   kubectl apply → AKS
```

---

## Layout

```
fleetops-gitops/
├── apps/
│   ├── fleet-api/              Helm chart — FastAPI service
│   ├── telemetry-worker/       Helm chart — GPS simulation worker
│   └── postgres/               Helm chart — PostgreSQL StatefulSet
├── argocd/                     ArgoCD Application manifests
│   ├── fleet-api-app.yaml
│   ├── telemetry-worker-app.yaml
│   ├── ingress-nginx-app.yaml
│   ├── prometheus-app.yaml
│   └── grafana-app.yaml
├── cluster/                    Cluster-wide config (applied once, not via ArgoCD)
│   ├── cluster-issuer.yaml     Let's Encrypt ClusterIssuer
│   └── ingress-nginx-values.yaml
└── grafana/
    └── dashboards/
        └── fleetops-api-overview.json   Grafana dashboard — persisted in repo
```

---

## Notes

- `postgres` is managed as a native Kubernetes StatefulSet with `priorityClassName: system-cluster-critical`. It is **not** an ArgoCD Application — see [ADR-001](https://github.com/daiberia/fleetops/blob/main/docs/adr/) for context on the eviction issue that drove this decision.
- `ingress-nginx` ArgoCD Application shows `Sync: Unknown` — the Helm release was bootstrapped manually before ArgoCD took ownership. nginx is fully operational; this is a cosmetic status only.
- Grafana has no persistent volume. The dashboard JSON in `grafana/dashboards/` is the source of truth and is loaded on pod start.
