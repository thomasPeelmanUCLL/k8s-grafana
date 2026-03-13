# k8s-grafana

GitOps-style deployment of [Grafana](https://grafana.com) on Kubernetes via Helm.

## Features

- Installs Grafana via the official `grafana/grafana` Helm chart
- GitHub Actions workflow for manual deploy (`workflow_dispatch`)
- Weekly auto-update check that opens a PR when a new chart version is available
- Persistent storage via PVC
- Ingress with TLS via cert-manager (Traefik)
- Pre-configured datasource for Prometheus

## Repository Structure

```
.
├── .github/
│   └── workflows/
│       ├── deploy.yml          # Manual deploy
│       └── update-check.yml    # Weekly version check
├── helm/
│   └── values.yaml             # All Grafana Helm values
├── dashboards/
│   └── cert-manager.json       # Example pre-provisioned dashboard
├── docs/
│   └── wiki.md                 # Full ops & config guide
└── README.md
```

## Quick Start

### Prerequisites

- `kubectl` configured against your cluster
- `helm` v3 installed
- cert-manager installed (for TLS) — see [k8s-cert-manager](https://github.com/thomasPeelmanUCLL/k8s-cert-manager)
- GitHub Actions secret `KUBECONFIG` set

### Manual Deploy via GitHub Actions

1. Go to **Actions → Deploy Grafana**
2. Click **Run workflow**
3. Optionally pin a chart version (blank = latest)

### Local Deploy

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace \
  -f helm/values.yaml
```

## Updating

Every Monday at 08:00 UTC the `update-check.yml` workflow compares the latest chart version to the pinned value in `helm/values.yaml`. If a newer version exists it opens a PR — review and merge manually.

## Default Credentials

The admin password is stored in a Kubernetes secret. Retrieve it with:

```bash
kubectl get secret grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode
```

Or set a custom password in `helm/values.yaml` under `adminPassword`.

## License

MIT
