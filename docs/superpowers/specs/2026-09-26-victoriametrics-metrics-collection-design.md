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
- **Layout:** `infrastructure/` gets the same base + overlay layout as `apps/`
  and `databases/`, because more environments are expected.

## Components

Namespace: `monitoring`.

| Component | State | Reason |
|---|---|---|
| VictoriaMetrics operator + CRDs | on | Manages VMSingle and VMAgent |
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

The rule is the standard `kubernetes-pods` scrape job, set in
`vmagent.spec.inlineScrapeConfig` in the HelmRelease values. Pods without
`prometheus.io/scrape: "true"` are not scraped by this rule. Each annotated pod
must appear exactly once as a target.

## App changes

Add the three annotations (port `8000`, path `/metrics`) to the pod template in:

- `apps/base/ml-api/deployment.yaml`
- `apps/base/backend-api/deployment.yaml`

Flux controllers already carry the annotations. Postgres and load-generator
expose no metrics endpoint; cAdvisor and kube-state-metrics cover them.

## Folder layout

```
infrastructure/
  base/
    controllers/victoria-metrics/   namespace, HelmRepository, HelmRelease
  devops-cs/
    controllers/
      kustomization.yaml            lists victoria-metrics
      victoria-metrics/             kustomization.yaml -> ../../../base/controllers/victoria-metrics
    configs/
      kustomization.yaml            empty (resources: [])
```

- Remove the empty `infrastructure/controllers/`, `infrastructure/configs/` and
  the unused `infrastructure/kustomization.yaml`.
- `clusters/devops-cs/infrastructure.yaml`: `infra-controllers` path becomes
  `./infrastructure/devops-cs/controllers`, `infra-configs` path becomes
  `./infrastructure/devops-cs/configs`.

## Flux ordering

`infra-controllers` installs everything; `databases` and `apps` don't depend on
it.

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
