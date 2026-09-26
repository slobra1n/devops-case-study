# VictoriaMetrics Metrics Collection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Collect metrics from every pod in the cluster into VictoriaMetrics with one annotation-based scrape rule.

**Architecture:** Monitoring is its own layer under `infrastructure/monitoring/`, shared by all clusters, with two Flux Kustomizations in `clusters/devops-cs/monitoring.yaml`: `monitoring-controllers` installs the trimmed `victoria-metrics-k8s-stack` Helm chart; `monitoring-configs` then applies one `VMPodScrape` that implements the `prometheus.io/*` annotations, as documented by the VictoriaMetrics operator. ml-api and backend-api opt in with those annotations. Nothing depends on monitoring.

**Tech Stack:** Flux v2.9.5 (`helm.toolkit.fluxcd.io/v2`, `source.toolkit.fluxcd.io/v1`), Kustomize via `kubectl kustomize`, `victoria-metrics-k8s-stack` 0.93.0, k3d/k3s.

**Spec:** [VictoriaMetrics metrics collection](../specs/2026-09-26-victoriametrics-metrics-collection-design.md)

## Global Constraints

- Namespace: `monitoring`.
- Chart: `victoria-metrics-k8s-stack` version `0.93.0` from `https://victoriametrics.github.io/helm-charts/` (latest release, 2026-09-20).
- VMSingle: 5Gi PVC on the cluster's default StorageClass (`local-path`), `retentionPeriod: "14d"`.
- Off: Grafana, Alertmanager, vmalert, default rules, default dashboards, node-exporter, API server, controller-manager, scheduler, etcd.
- Scrape rule: one `VMPodScrape` `monitoring/annotations-discovery` in `monitoring-configs`, based on the VictoriaMetrics operator's [annotation auto-discovery example](https://docs.victoriametrics.com/operator/integrations/prometheus/); no `inlineScrapeConfig`.
- Layout: `infrastructure/monitoring/{controllers,configs}/victoria-metrics/`, shared by all clusters (no per-cluster overlay). Per-cluster differences are `patches:` in `clusters/<cluster>/monitoring.yaml`. `infrastructure/controllers` and `infrastructure/configs` stay as empty placeholders on their original paths.
- `databases` and `apps` do not depend on monitoring. No ingress.
- No application code changes. Only pod-template annotations in `apps/base`.
- No `git push` in this plan. The user pushes; Task 3 runs after that.

## Review Focus

1. An undeclared port, a multi-port pod or an annotated chart pod shows up as a missing or extra target. Task 3 checks exactly 8 pods, each scraped once.
2. CRDs missing on install or stale after upgrades. The chart installs them (Flux default `Create`), the HelmRelease sets `upgrade.crds: CreateReplace`, and `monitoring-configs` waits for `monitoring-controllers` before applying the `VMPodScrape`. Task 3 checks every Kustomization and the HelmRelease are Ready.
3. Metrics lost on VMSingle restart. Task 3 checks the PVC is Bound and data from before a pod deletion is still visible.

---

## Task 1: Monitoring layer with the VictoriaMetrics stack

**Files:**
- Delete: `infrastructure/kustomization.yaml` (unused)
- Create: `infrastructure/monitoring/controllers/kustomization.yaml`
- Create: `infrastructure/monitoring/controllers/victoria-metrics/namespace.yaml`
- Create: `infrastructure/monitoring/controllers/victoria-metrics/helmrepository.yaml`
- Create: `infrastructure/monitoring/controllers/victoria-metrics/helmrelease.yaml`
- Create: `infrastructure/monitoring/controllers/victoria-metrics/kustomization.yaml`
- Create: `infrastructure/monitoring/configs/kustomization.yaml`
- Create: `infrastructure/monitoring/configs/victoria-metrics/vmpodscrape.yaml`
- Create: `infrastructure/monitoring/configs/victoria-metrics/kustomization.yaml`
- Create: `clusters/devops-cs/monitoring.yaml`

- [ ] **Step 1: Remove the unused top-level kustomization**

```sh
git rm -q infrastructure/kustomization.yaml
```

- [ ] **Step 2: Write the definitions**

`infrastructure/monitoring/controllers/victoria-metrics/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

`infrastructure/monitoring/controllers/victoria-metrics/helmrepository.yaml`:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: victoriametrics
  namespace: monitoring
spec:
  interval: 24h
  url: https://victoriametrics.github.io/helm-charts/
```

`infrastructure/monitoring/controllers/victoria-metrics/helmrelease.yaml`:

```yaml
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: victoria-metrics-k8s-stack
  namespace: monitoring
spec:
  interval: 10m
  chart:
    spec:
      chart: victoria-metrics-k8s-stack
      version: "0.93.0"
      sourceRef:
        kind: HelmRepository
        name: victoriametrics
  upgrade:
    crds: CreateReplace
  values:
    grafana:
      enabled: false
    alertmanager:
      enabled: false
    vmalert:
      enabled: false
    defaultRules:
      enabled: false
    defaultDashboards:
      enabled: false
    prometheus-node-exporter:
      enabled: false
    kubeApiServer:
      enabled: false
    kubeControllerManager:
      enabled: false
    kubeScheduler:
      enabled: false
    kubeEtcd:
      enabled: false
    vmsingle:
      spec:
        retentionPeriod: "14d"
        storage:
          resources:
            requests:
              storage: 5Gi
```

`infrastructure/monitoring/controllers/victoria-metrics/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - helmrepository.yaml
  - helmrelease.yaml
```

`infrastructure/monitoring/configs/victoria-metrics/vmpodscrape.yaml`:

```yaml
# Based on the VictoriaMetrics operator docs, "Auto-discovery for
# prometheus.io annotations":
# https://docs.victoriametrics.com/operator/integrations/prometheus/
apiVersion: operator.victoriametrics.com/v1beta1
kind: VMPodScrape
metadata:
  name: annotations-discovery
  namespace: monitoring
spec:
  # Every pod in every namespace; the relabel rules keep only annotated ones.
  namespaceSelector:
    any: true
  selector: {}
  podMetricsEndpoints:
    - relabelConfigs:
        - action: drop
          source_labels: [__meta_kubernetes_pod_container_init]
          regex: "true"
        - action: keep_if_equal
          source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port, __meta_kubernetes_pod_container_port_number]
        - action: keep
          source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
          regex: "true"
        # regex (.+) keeps the default /metrics when the path annotation is
        # absent (the Flux controllers don't set it).
        - action: replace
          source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
          target_label: __metrics_path__
          regex: (.+)
        - action: replace
          source_labels: [__meta_kubernetes_pod_node_name]
          target_label: node
```

`infrastructure/monitoring/configs/victoria-metrics/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - vmpodscrape.yaml
```

- [ ] **Step 3: List the components per step**

`infrastructure/monitoring/controllers/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - victoria-metrics
```

`infrastructure/monitoring/configs/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - victoria-metrics
```

- [ ] **Step 4: Add the Flux Kustomizations**

`clusters/devops-cs/monitoring.yaml`:

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: monitoring-controllers
  namespace: flux-system
spec:
  interval: 10m
  retryInterval: 1m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/monitoring/controllers
  prune: true
  wait: true
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: monitoring-configs
  namespace: flux-system
spec:
  dependsOn:
    - name: monitoring-controllers
  interval: 10m
  retryInterval: 1m
  sourceRef:
    kind: GitRepository
    name: flux-system
  path: ./infrastructure/monitoring/configs
  prune: true
  wait: true
```

`clusters/devops-cs/infrastructure.yaml` keeps its original paths (`./infrastructure/controllers`, `./infrastructure/configs`).

- [ ] **Step 5: Verify the render and the isolation**

```sh
kubectl kustomize infrastructure/monitoring/controllers | grep -E '^kind:'
kubectl kustomize infrastructure/monitoring/configs | grep -E '^kind:'
grep -l monitoring clusters/devops-cs/apps.yaml clusters/devops-cs/databases.yaml clusters/devops-cs/infrastructure.yaml
```

Expected: `kind: Namespace`, `kind: HelmRepository`, `kind: HelmRelease`; then `kind: VMPodScrape`; the `grep -l` prints nothing (nothing depends on monitoring).

- [ ] **Step 6: Commit**

```sh
git add infrastructure clusters/devops-cs/monitoring.yaml
git commit -m "feat: add VictoriaMetrics as infrastructure/monitoring layer"
```

---

## Task 2: Annotate ml-api and backend-api

**Files:**
- Modify: `apps/base/ml-api/deployment.yaml` (pod template `metadata`)
- Modify: `apps/base/backend-api/deployment.yaml` (pod template `metadata`)

- [ ] **Step 1: Add the annotations**

In both files, under `spec.template.metadata`, next to `labels`:

```yaml
  template:
    metadata:
      labels:
        app: ml-api            # backend-api in the other file
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
```

- [ ] **Step 2: Verify**

Run: `kubectl kustomize apps/devops-cs | grep -c 'prometheus.io/scrape: "true"'`
Expected: `2`.

- [ ] **Step 3: Commit**

```sh
git add apps/base/ml-api/deployment.yaml apps/base/backend-api/deployment.yaml
git commit -m "feat: opt ml-api and backend-api into metrics scraping"
```

---

## Task 3: Verify after the user pushes

The user runs `git pull --rebase` (to pick up the two Flux bootstrap commits on `origin/main`) and `git push`. That push deploys the pending `apps/base` + `databases/` refactor together with Tasks 1 and 2: postgres starts with an empty database, and backend-api starts after it because `apps` waits for `databases`.

**Files:** none changed.

- [ ] **Step 1: Flux is Ready (criterion 1)**

```sh
flux get kustomizations
flux get helmreleases -A
kubectl -n monitoring get pods,pvc
```

Expected: every Kustomization and `monitoring/victoria-metrics-k8s-stack` `True`; pods Running; PVC `Bound`, `5Gi`. The first install can take a few minutes while images pull.

- [ ] **Step 2: Backend still serves 200s**

```sh
kubectl logs -n backend-api -l app=backend-api --since=2m --prefix | grep 'POST /process' | tail -5
```

Expected: `200 OK`. If `500`, apply the known workaround from `TEMP-NOTES.md`: `kubectl -n backend-api rollout restart deployment/backend-api`, wait for the rollout, re-run the check.

- [ ] **Step 3: Open the VictoriaMetrics web pages**

```sh
kubectl -n monitoring get svc
kubectl -n monitoring port-forward svc/vmagent-victoria-metrics-k8s-stack 8429 &
kubectl -n monitoring port-forward svc/vmsingle-victoria-metrics-k8s-stack 8428 &
```

If the service names differ, use the ones `get svc` lists. Give VMAgent a minute to scrape.

- [ ] **Step 4: Targets (criterion 2)**

Open http://localhost:8429/targets.

Expected: every target `up`. The `monitoring/annotations-discovery` group (the operator sets `job` to `<namespace>/<VMPodScrape name>`) has exactly 8 targets: 2 `ml-api`, 2 `backend-api`, 4 `flux-system` controllers. Groups for kubelet, kube-state-metrics, CoreDNS and the VictoriaMetrics components are also present. Any extra or missing pod: see Review Focus 1.

- [ ] **Step 5: Data (criterion 3)**

Open http://localhost:8428/vmui and run each query:

- `backend_api_requests_total`
- `container_memory_working_set_bytes{namespace="postgres"}`
- `kube_pod_container_status_restarts_total`

Expected: each returns data.

- [ ] **Step 6: Data survives a VMSingle restart (criterion 4)**

In vmui, set the time range to "Last 1 hour" and graph `count(up)`. Then:

```sh
kubectl -n monitoring delete pod -l app.kubernetes.io/name=vmsingle
kubectl -n monitoring wait --for=condition=Ready pod -l app.kubernetes.io/name=vmsingle --timeout=180s
kubectl -n monitoring port-forward svc/vmsingle-victoria-metrics-k8s-stack 8428 &
```

The old port-forward breaks when its pod is deleted, so the last line starts a new one. Reload vmui.

Expected: the graph still shows data from before the deletion.

- [ ] **Step 7: Clean up and report**

Stop the port-forwards (`kill %1 %2 %3`, or close the shell) and report the results of Steps 1–6 to the user. No commit: this task changes no files.
