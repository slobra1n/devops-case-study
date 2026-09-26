# VictoriaMetrics Metrics Collection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Collect metrics from every pod in the cluster into VictoriaMetrics with one annotation-based scrape rule.

**Architecture:** Flux installs the trimmed `victoria-metrics-k8s-stack` Helm chart through the `infra-controllers` Kustomization, from a new base + overlay layout under `infrastructure/`. VMAgent gets the standard `kubernetes-pods` job inline; ml-api and backend-api opt in with `prometheus.io/*` pod annotations.

**Tech Stack:** Flux v2.9.5 (`helm.toolkit.fluxcd.io/v2`, `source.toolkit.fluxcd.io/v1`), Kustomize via `kubectl kustomize`, `victoria-metrics-k8s-stack` 0.93.0, k3d/k3s.

**Spec:** [VictoriaMetrics metrics collection](../specs/2026-09-26-victoriametrics-metrics-collection-design.md)

## Global Constraints

- Namespace: `monitoring`.
- Chart: `victoria-metrics-k8s-stack` version `0.93.0` from `https://victoriametrics.github.io/helm-charts/` (latest release, 2026-09-20).
- VMSingle: 5Gi PVC on the cluster's default StorageClass (`local-path`), `retentionPeriod: "14d"`.
- Off: Grafana, Alertmanager, vmalert, default rules, default dashboards, node-exporter, API server, controller-manager, scheduler, etcd.
- Scrape rule: standard `kubernetes-pods` job in `vmagent.spec.inlineScrapeConfig`; no `VMPodScrape`.
- Layout: `infrastructure/base/...` + `infrastructure/devops-cs/...`, one folder per component, same as `apps/` and `databases/`.
- `databases` and `apps` do not depend on monitoring. No ingress.
- No application code changes. Only pod-template annotations in `apps/base`.
- No `git push` in this plan. The user pushes; Task 3 runs after that.

## Review Focus

1. An undeclared port, a multi-port pod or an annotated chart pod shows up as a missing or extra target. Task 3 checks exactly 8 pods, each scraped once.
2. CRDs missing on install or stale after upgrades. The chart installs them (Flux default `Create`) and the HelmRelease sets `upgrade.crds: CreateReplace`; Task 3 checks the HelmRelease is Ready.
3. Metrics lost on VMSingle restart. Task 3 checks the PVC is Bound and data from before a pod deletion is still visible.

---

## Task 1: Infrastructure base + overlay with the VictoriaMetrics HelmRelease

**Files:**
- Delete: `infrastructure/kustomization.yaml`, `infrastructure/controllers/kustomization.yaml`, `infrastructure/configs/kustomization.yaml`
- Create: `infrastructure/base/controllers/victoria-metrics/namespace.yaml`
- Create: `infrastructure/base/controllers/victoria-metrics/helmrepository.yaml`
- Create: `infrastructure/base/controllers/victoria-metrics/helmrelease.yaml`
- Create: `infrastructure/base/controllers/victoria-metrics/kustomization.yaml`
- Create: `infrastructure/devops-cs/controllers/kustomization.yaml`
- Create: `infrastructure/devops-cs/controllers/victoria-metrics/kustomization.yaml`
- Modify: `clusters/devops-cs/infrastructure.yaml` (`infra-controllers` path; remove `infra-configs`)

- [ ] **Step 1: Remove the old empty layout**

```sh
git rm -q infrastructure/kustomization.yaml infrastructure/controllers/kustomization.yaml infrastructure/configs/kustomization.yaml
```

- [ ] **Step 2: Write the base**

`infrastructure/base/controllers/victoria-metrics/namespace.yaml`:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
```

`infrastructure/base/controllers/victoria-metrics/helmrepository.yaml`:

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

`infrastructure/base/controllers/victoria-metrics/helmrelease.yaml`:

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
    vmagent:
      spec:
        # Standard annotation-based pod discovery, copied from the
        # victoria-metrics-agent chart's default config.
        inlineScrapeConfig: |
          - job_name: kubernetes-pods
            kubernetes_sd_configs:
              - role: pod
            relabel_configs:
              - action: drop
                source_labels: [__meta_kubernetes_pod_container_init]
                regex: true
              - action: keep_if_equal
                source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port, __meta_kubernetes_pod_container_port_number]
              - action: keep
                source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
                regex: true
              - action: replace
                source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_path]
                target_label: __metrics_path__
                regex: (.+)
              - action: replace
                source_labels: [__address__, __meta_kubernetes_pod_annotation_prometheus_io_port]
                regex: ([^:]+)(?::\d+)?;(\d+)
                replacement: $1:$2
                target_label: __address__
              - action: labelmap
                regex: __meta_kubernetes_pod_label_(.+)
              - source_labels: [__meta_kubernetes_pod_name]
                target_label: pod
              - source_labels: [__meta_kubernetes_pod_container_name]
                target_label: container
              - source_labels: [__meta_kubernetes_namespace]
                target_label: namespace
              - source_labels: [__meta_kubernetes_pod_node_name]
                target_label: node
```

`infrastructure/base/controllers/victoria-metrics/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - namespace.yaml
  - helmrepository.yaml
  - helmrelease.yaml
```

- [ ] **Step 3: Write the overlay**

`infrastructure/devops-cs/controllers/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - victoria-metrics
```

`infrastructure/devops-cs/controllers/victoria-metrics/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - ../../../base/controllers/victoria-metrics
```

- [ ] **Step 4: Point Flux at the overlay**

In `clusters/devops-cs/infrastructure.yaml`:
- `infra-controllers`: `path: ./infrastructure/controllers` → `path: ./infrastructure/devops-cs/controllers`
- Delete the `infra-configs` document (the `---` and everything after it). It applies nothing and nothing depends on it; add it back when the first config exists.

- [ ] **Step 5: Verify the render**

```sh
kubectl kustomize infrastructure/devops-cs/controllers | grep -E '^kind:'
kubectl kustomize infrastructure/devops-cs/controllers | python3 -c '
import sys, yaml
hr = next(d for d in yaml.safe_load_all(sys.stdin) if d and d["kind"] == "HelmRelease")
jobs = yaml.safe_load(hr["spec"]["values"]["vmagent"]["spec"]["inlineScrapeConfig"])
assert [j["job_name"] for j in jobs] == ["kubernetes-pods"]
print("scrape config parses")'
```

Expected: `kind: Namespace`, `kind: HelmRepository`, `kind: HelmRelease`; `scrape config parses`.

- [ ] **Step 6: Commit**

```sh
git add infrastructure clusters/devops-cs/infrastructure.yaml
git commit -m "feat: add VictoriaMetrics stack under infrastructure base/overlay"
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

Expected: every target `up`. The `kubernetes-pods` group has exactly 8 targets: 2 `ml-api`, 2 `backend-api`, 4 `flux-system` controllers. Groups for kubelet, kube-state-metrics, CoreDNS and the VictoriaMetrics components are also present. Any extra or missing pod: see Review Focus 1.

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
