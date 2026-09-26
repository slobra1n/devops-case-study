# VictoriaMetrics metrics collection

## Goal

Collect metrics from every pod in the cluster into VictoriaMetrics with one
scrape rule. This step collects and stores metrics only.

## Decisions

- **Stack:** `victoria-metrics-k8s-stack` Helm chart from
  `https://victoriametrics.github.io/helm-charts/`, with unneeded parts turned
  off. Pin an exact chart version (picked in the plan).
- **Discovery:** pods opt in with the `prometheus.io/*` annotations. This is a
  widely used convention, not an official Kubernetes standard. One rule covers
  every pod, so VictoriaMetrics needs no per-app configuration.
- **Storage:** VMSingle on a 5Gi PVC (`local-path` StorageClass), 14 days
  retention. Metrics must survive pod and cluster restarts.
- **Layout:** every layer (`infrastructure/`, `databases/`, `apps/`) has the
  same shape: `<layer>/base/<component>/` holds the definition, written once;
  `<layer>/<cluster>/<component>/` picks it for that cluster and holds that
  cluster's differences; `clusters/<cluster>/<layer>.yaml` is Flux wiring only.
  Monitoring is the component `infrastructure/base/monitoring/`.
- **Multi-cluster, no repetition:** definitions exist once in `base/`. A cluster
  that needs something different (e.g. another chart version) patches it in its
  own `<layer>/<cluster>/<component>/kustomization.yaml`.

## Components

Namespace: `monitoring`.

| Component | State | Reason |
|---|---|---|
| VictoriaMetrics operator + CRDs | on | Manages VMSingle, VMAgent and scrape objects |
| VMAgent | on | Scrapes all targets |
| VMSingle | on | Stores metrics |
| kube-state-metrics | on | Restarts, readiness and replicas for every pod |
| kubelet / cAdvisor scrape | on | CPU and memory for every container |
| CoreDNS scrape | on | Cluster DNS |
| node-exporter | off | Node (Docker VM) metrics not needed; cAdvisor covers containers |
| Grafana, Alertmanager, vmalert, default rules, default dashboards | off | Out of scope |
| controller-manager, scheduler, etcd scrapes | off | Embedded in the k3s process; targets would always fail |
| API server scrape | off | High volume, not needed for app monitoring |

## Annotation rule

VMAgent scrapes pods in all namespaces that have:

```yaml
prometheus.io/scrape: "true"
prometheus.io/port: "<port>"
prometheus.io/path: "/metrics"   # optional, defaults to /metrics
```

The rule is one `VMPodScrape` named `annotations-discovery` in `monitoring`,
following the VictoriaMetrics operator's documented
[Auto-discovery for prometheus.io annotations](https://docs.victoriametrics.com/operator/integrations/prometheus/)
example. It ships inside the HelmRelease through the chart's `extraObjects`
value: Helm installs the chart's CRDs before its other objects, so it needs no
separate Flux step (the chart creates VMAgent and VMSingle the same way). Pods
without `prometheus.io/scrape: "true"` are not scraped by this rule. Each
annotated pod must appear exactly once as a target.

## App changes

Add the three annotations (port `8000`, path `/metrics`) to the pod template in:

- `apps/base/ml-api/deployment.yaml`
- `apps/base/backend-api/deployment.yaml`

Flux controllers already carry the annotations. Postgres and load-generator
expose no metrics endpoint; cAdvisor and kube-state-metrics cover them.

## Folder layout

```
clusters/devops-cs/                 Flux wiring only
  infrastructure.yaml               infrastructure → ./infrastructure/devops-cs
  databases.yaml                    databases      → ./databases/devops-cs
  apps.yaml                         apps           → ./apps/devops-cs (waits for databases)
infrastructure/
  base/monitoring/                  namespace, HelmRepository, HelmRelease (incl. VMPodScrape)
  devops-cs/
    kustomization.yaml              lists monitoring
    monitoring/kustomization.yaml   → ../../base/monitoring
databases/  base/postgres/  devops-cs/postgres/
apps/       base/<app>/     devops-cs/<app>/
```

- Remove `infrastructure/controllers/`, `infrastructure/configs/` and the unused
  `infrastructure/kustomization.yaml`, and the `infra-controllers` /
  `infra-configs` Flux Kustomizations. `databases` and `apps` drop their
  `dependsOn: infra-controllers`.
- A new infrastructure component (cert-manager, Loki) is a new
  `infrastructure/base/<component>/`, a new
  `infrastructure/<cluster>/<component>/`, and one line in
  `infrastructure/<cluster>/kustomization.yaml`. `clusters/` does not change.

## Flux ordering

`infrastructure` applies the monitoring HelmRelease; `databases` then `apps`
(`dependsOn: databases`). Nothing depends on `infrastructure`, so a failing
monitoring install never blocks `databases` or `apps`. When an app-critical
component such as cert-manager is added, decide then whether apps wait for all
of `infrastructure` or that component gets its own Flux Kustomization.

## Per-cluster differences

Add a patch to that cluster's overlay, e.g.
`infrastructure/<cluster>/monitoring/kustomization.yaml` for a different chart
version:

```yaml
resources:
  - ../../base/monitoring
patches:
  - target:
      kind: HelmRelease
      name: victoria-metrics-k8s-stack
    patch: |
      - op: replace
        path: /spec/chart/spec/version
        value: "0.92.1"
```

## Access

No ingress. Open vmui with `kubectl port-forward` to the VMSingle service.

## Acceptance criteria

1. `flux get kustomizations` and `flux get helmreleases -A` show all Ready.
2. VMAgent's target list shows every target up, and each annotated pod exactly
   once: 2 ml-api, 2 backend-api, 4 Flux controllers. Kubelet/cAdvisor,
   kube-state-metrics, CoreDNS and VictoriaMetrics' own components are also up.
3. These queries return data:
   - `backend_api_requests_total`
   - `container_memory_working_set_bytes{namespace="postgres"}`
   - `kube_pod_container_status_restarts_total`
4. After deleting the VMSingle pod, data from before the deletion is still
   queryable.

## Known limits

- Annotations aren't checked. A typo or wrong port means the pod silently isn't
  scraped. Check the target list or `up`.
- `local-path` storage is tied to the single k3d node.

## Out of scope

Grafana, alerting, SLOs, postgres exporter, logging, ingress, high availability.
