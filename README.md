# helm-catalog

Shared Helm chart catalog. **Installed by Argo CD automatically** — you do not `helm install` these directly. Hub ApplicationSet and spoke child Applications reference charts here by `repoURL` + `path`.

Each subdirectory is a standalone Helm chart. Reference it with `repoURL` pointing to this repo's remote and `path: <chart-folder-name>`.

| Chart | Installed by | Target cluster | Description |
|-------|-------------|----------------|-------------|
| `spoke-argocd-bootstrap` | Hub ApplicationSet (Argo CD) | Each ROSA spoke | Installs GitOps operator + creates root app-of-apps |
| `rosa-machine-pool` | Spoke child Application (Argo CD) | Spoke (in-cluster) | Day-2 ROSA MachinePool (2 replicas example) |
| `cluster-banner` | Spoke/hub child Application (Argo CD) | All clusters | OCP ConsoleNotification banner colour-coded by environment |
| `alertmanager` | Spoke/hub child Application (Argo CD) | All clusters | Prometheus Alertmanager wrapper (prometheus-community/alertmanager) |
| `stackrox` | Hub child Application (Argo CD) | Hub only | RHACS Central Services wrapper (charts.stackrox.io/central-services) |
| `devspaces` | Spoke child Application (Argo CD) | Dev spokes | Red Hat DevSpaces CheCluster CR — browser-based VS Code workspaces for dev teams |

## Local render (dry-run / review)

```bash
# Bootstrap chart
helm template spoke spoke-argocd-bootstrap/ \
  --set clusterType=dev,clusterGroup=eng,region=eu-west-1,platform.version=main

# ROSA MachinePool chart
helm template rosa rosa-machine-pool/ \
  --set rosa.machinePools.platform.enabled=true \
  --set rosa.machinePools.platform.clusterID=<your-cluster-id>
```

## Adding a chart

1. Create `helm-catalog/<chart-name>/` with `Chart.yaml`, `values.yaml`, `templates/`.
2. Add a catalog entry to this README.
3. Reference from `platform-config/default.yaml` with `repoURL: https://github.com/<org>/helm-catalog.git` and `path: <chart-name>`, or directly in a hub ApplicationSet.
