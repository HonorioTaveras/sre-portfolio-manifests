# SRE Portfolio -- Kubernetes Manifests

GitOps repository for the SRE Portfolio project. Contains Helm charts and ArgoCD
Application manifests. This repository is the source of truth for all Kubernetes
deployments -- no manual `kubectl apply` or `helm install` in production.

## Repository Structure

```
charts/
├── api/                    # Helm chart for the FastAPI API service
│   ├── Chart.yaml
│   ├── values.yaml         # Default values
│   ├── values-dev.yaml     # Dev environment overrides
│   ├── values-prod.yaml    # Production overrides
│   └── templates/
│       ├── deployment.yaml
│       ├── service.yaml
│       └── serviceaccount.yaml
├── worker/                 # Helm chart for the background worker service
│   └── ...
├── argocd/                 # ArgoCD Application manifests
│   ├── api-app.yaml
│   └── worker-app.yaml
└── observability/          # Observability stack configuration
    ├── datadog-values.yaml
    ├── otel-collector.yaml
    ├── servicemonitor.yaml
    └── grafana-dashboard-sre-portfolio.json
```

## GitOps Flow

```
Developer pushes code to sre-portfolio-app
        │
        └── GitHub Actions pipeline
                ├── Builds Docker image
                ├── Pushes to ECR with git SHA tag
                └── Updates image tag in this repo (values-dev.yaml)
                        │
                        └── ArgoCD detects change
                                └── Syncs cluster to new image tag
```

## ArgoCD Applications

Both applications are configured with:
- `selfHeal: true` -- manual cluster changes are automatically reverted
- `prune: true` -- resources removed from Git are removed from the cluster
- Automated sync -- no manual sync required

## Helm Charts

### API Service
FastAPI job submission service. Key configuration:
- Rolling update: `maxUnavailable: 0`, `maxSurge: 1` for zero-downtime deploys
- Readiness probe on `/health` controls traffic routing
- Liveness probe on `/health` controls pod restarts
- Prometheus annotations for automatic metrics scraping
- IRSA-ready ServiceAccount for AWS permissions

### Worker Service
Background job processor. Same rolling update strategy. Uses ddtrace for Datadog APM.

## Observability Stack

Deployed to the `monitoring` namespace:

| Component | Purpose |
|---|---|
| kube-prometheus-stack | Prometheus + Grafana + Alertmanager |
| Grafana Tempo | Distributed trace storage |
| OTel Collector | Telemetry pipeline -- receives traces, fans out to Tempo and Datadog |
| Datadog Agent | Infrastructure metrics + APM |
| ServiceMonitor | Prometheus autodiscovery for app services |

## Related Repository

Application code, Terraform infrastructure, and CI/CD pipeline:
[sre-portfolio-app](https://github.com/HonorioTaveras/sre-portfolio-app)
