# Grafana GitOps Wiki

> This file mirrors the GitHub Wiki. Keep both in sync.

---

## Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Secrets Setup](#secrets)
4. [Deploying](#deploying)
5. [Accessing Grafana](#accessing-grafana)
6. [Updating](#updating)
7. [Helm Values Reference](#helm-values-reference)
8. [Pre-provisioned Dashboards](#pre-provisioned-dashboards)
9. [Troubleshooting](#troubleshooting)
10. [Useful Commands](#useful-commands)

---

## Overview

This repository manages the lifecycle of [Grafana](https://grafana.com) on a Kubernetes cluster using Helm and GitHub Actions. Grafana is deployed in the `monitoring` namespace alongside a Prometheus datasource.

---

## Prerequisites

| Tool | Minimum version |
|------|-----------------|
| Kubernetes | 1.25+ |
| Helm | 3.x |
| cert-manager | Installed (for TLS ingress) |
| Traefik | Installed as ingress controller |
| Prometheus | Running in cluster (optional but recommended) |

---

## Secrets

Go to **Settings → Secrets and variables → Actions** and add:

| Secret name | Description |
|-------------|-------------|
| `KUBECONFIG` | Full kubeconfig for your cluster |

For K3s:
```bash
cat /etc/rancher/k3s/k3s.yaml
# Replace 127.0.0.1 with your cluster public IP
```

---

## Deploying

### Before deploying

1. Edit `helm/values.yaml` and replace all `yourdomain.com` with your actual domain
2. Set `adminPassword` or leave empty for auto-generated
3. Make sure cert-manager and your `letsencrypt-prod` ClusterIssuer are ready

### Via GitHub Actions

1. **Actions → Deploy Grafana → Run workflow**
2. Optionally enter a chart version
3. Leave blank for latest

### Locally

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm upgrade --install grafana grafana/grafana \
  --namespace monitoring \
  --create-namespace \
  -f helm/values.yaml
```

---

## Accessing Grafana

### Via Ingress (TLS)

Once deployed, Grafana is available at `https://grafana.yourdomain.com`.

### Retrieve admin password

```bash
kubectl get secret grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode && echo
```

### Port-forward (no ingress needed)

```bash
kubectl port-forward svc/grafana 3000:80 -n monitoring
# Open http://localhost:3000
```

---

## Updating

Version priority:

1. **Workflow dispatch input** — overrides everything
2. **`chartVersion` in values.yaml** — used if non-empty
3. **Live latest** — resolved from Helm repo

The `update-check.yml` runs weekly and opens a PR when pinned version is outdated. Review the [Grafana changelog](https://github.com/grafana/grafana/releases) before merging.

---

## Helm Values Reference

| Key | Default | Description |
|-----|---------|-------------|
| `chartVersion` | `""` | Pin chart version; empty = latest |
| `adminUser` | `admin` | Grafana admin username |
| `adminPassword` | `""` | Admin password; empty = auto-generated |
| `persistence.enabled` | `true` | Enable PVC for dashboards/data |
| `persistence.size` | `5Gi` | PVC size |
| `ingress.enabled` | `true` | Enable Ingress |
| `ingress.ingressClassName` | `traefik` | Ingress class |
| `datasources` | Prometheus | Auto-provisioned datasource |

---

## Pre-provisioned Dashboards

These dashboards are auto-downloaded from Grafana.com on deploy:

| Dashboard | Grafana ID | What it shows |
|-----------|-----------|---------------|
| cert-manager | 20340 | Certificate expiry, ACME requests |
| Kubernetes cluster | 7249 | Node/pod/namespace overview |
| Node Exporter Full | 1860 | CPU, memory, disk, network per node |

Add more by appending to `dashboards.default` in `helm/values.yaml` with any `gnetId` from [grafana.com/dashboards](https://grafana.com/grafana/dashboards).

---

## Troubleshooting

### Pod not starting

```bash
kubectl describe pod -n monitoring -l app.kubernetes.io/name=grafana
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana
```

### PVC pending

```bash
kubectl get pvc -n monitoring
kubectl describe pvc -n monitoring
# Check your storageClass supports dynamic provisioning
```

### Ingress not getting TLS cert

```bash
kubectl get certificate -n monitoring
kubectl describe certificate grafana-tls -n monitoring
# Make sure letsencrypt-prod ClusterIssuer is Ready
kubectl get clusterissuer
```

---

## Useful Commands

```bash
# Check Grafana pod status
kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana

# Get admin password
kubectl get secret grafana -n monitoring \
  -o jsonpath="{.data.admin-password}" | base64 --decode && echo

# List all Helm releases in monitoring namespace
helm list -n monitoring

# Show current deployed values
helm get values grafana -n monitoring

# Uninstall Grafana (keeps PVC)
helm uninstall grafana -n monitoring

# Delete PVC (WARNING: deletes all dashboard data)
kubectl delete pvc -n monitoring -l app.kubernetes.io/name=grafana

# Restart Grafana pod
kubectl rollout restart deployment/grafana -n monitoring
```
