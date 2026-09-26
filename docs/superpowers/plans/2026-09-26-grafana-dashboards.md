# Grafana Dashboards Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Grafana with 9 dashboards provisioned from git: Sloth's two SLO dashboards (fed by new Sloth metadata rules), our Apps and Platform boards, and five pinned upstream drill-downs.

**Architecture:** Grafana is switched on in the existing `victoria-metrics-k8s-stack` HelmRelease, stateless and without runtime downloads. Dashboard JSON files live in `infrastructure/base/monitoring/dashboards/`; Kustomize's `configMapGenerator` turns each into a labelled ConfigMap that Grafana's sidecar loads into the folder its annotation names. `scripts/slo-generate.sh` gains Sloth's metadata plugin, which the SLO dashboards read.

**Tech Stack:** `victoria-metrics-k8s-stack` 0.93.0 (Grafana 13.1.1, k8s-sidecar 2.8.1), Sloth v0.16.0, Kustomize `configMapGenerator`, Flux v2.9.5, python3 (throwaway scripts), Docker (`alpine/helm:3.17.3` for chart renders).

**Spec:** [Grafana dashboards](../specs/2026-09-26-grafana-dashboards-design.md)

Every file below was built and checked in a throwaway prototype first: the boards and the Sloth edits were loaded into a local Grafana 13.1.1 against the live VMSingle, and every dashboard opened without panel errors.

## Global Constraints

- Grafana from the chart: `grafana.enabled: true`; `grafana.ini` `plugins.preinstall_disabled: true`; sidecar `folderAnnotation: grafana_folder`, `provider.foldersFromFilesStructure: true`. `syncJob.enabled: false`; `defaultDashboards.enabled: false` stays. No persistence, no ingress; access via `kubectl port-forward`.
- One ConfigMap per dashboard file, named `dashboard-<file name without .json>`, namespace `monitoring`, label `grafana_dashboard: "1"`, annotation `grafana_folder: Overview|SLOs|Components`, no name hash suffix.
- Pins: Sloth grafana.com 14643 rev 2 and 14348 rev 5; flux2-monitoring-example `7ab65dc8b90f7a6751d88f18bbb4e1bee33bf334`; VictoriaMetrics `v1.152.0`; grafana.com 1860 rev 45; dotdc `v3.0.8`; postgres_exporter `v0.20.1`.
- Component dashboards are committed byte for byte as downloaded. Sloth dashboards get only the edits in Task 2 Step 3.
- Sloth plugin chain: `validate_victoria_metrics/v1`, `sli_rules/v1`, `metadata_rules/v1`. 4 SLOs × 15 = 60 recording rules, no alert rules.
- Our boards: UIDs `apps` and `platform`, `${datasource}` variable, queries as in the spec.
- No application code changes. Push only after the user approves it; Flux reads GitHub, so cluster checks (Task 4) start after the push.

## Review Focus

1. A provisioned Sloth dashboard that still references `${DS_PROMETHEUS}` fails on every panel ("data source not found"). Task 2 Step 3 greps for it and expects 0.
2. A dashboard ConfigMap without the label, without the folder annotation, or outside `monitoring` is ignored or lands in the wrong folder. Task 2 Step 5 and Task 3 Step 4 check all three in the render; Task 4 Step 3 checks the folders in Grafana.
3. After kube-state-metrics or VMSingle restart, their series come back with a new `pod` label, and a stat over the time range shows one value per old pod. Task 3's queries aggregate those series; Task 4 Step 4 checks the stats show one value each over 6 hours.
4. Our series have no `cluster` label; the pods drill-down filters on it. Task 4 Step 5 expects the ml-api pods to show.
5. `slo-rules.yaml` goes stale when the script's plugin chain changes. Task 1 Step 2 regenerates and checks the committed files contain the new rules.

---

## Task 1: Sloth metadata rules

**Files:**
- Modify: `scripts/slo-generate.sh`
- Modify (generated): `apps/base/backend-api/slo-rules.yaml`, `apps/base/ml-api/slo-rules.yaml`
- Modify: `docs/superpowers/specs/2026-09-26-slo-sli-design.md`

- [ ] **Step 1: Add the metadata plugin**

In `scripts/slo-generate.sh` replace

```bash
# 28-day SLO period, MetricsQL validation, and SLI recording rules only.
```

with

```bash
# 28-day SLO period, MetricsQL validation, SLI recording rules, and the metadata
# rules (objective, error budget, burn rate) the Sloth Grafana dashboards read.
```

and replace

```bash
    -s '{"id":"sloth.dev/core/sli_rules/v1"}' <"$spec")
```

with

```bash
    -s '{"id":"sloth.dev/core/sli_rules/v1"}' \
    -s '{"id":"sloth.dev/core/metadata_rules/v1"}' <"$spec")
```

- [ ] **Step 2: Regenerate and check**

```sh
scripts/slo-generate.sh
grep -c 'record:' apps/base/backend-api/slo-rules.yaml apps/base/ml-api/slo-rules.yaml
git diff --numstat -- apps/base
kubectl apply --dry-run=server -f apps/base/backend-api/slo-rules.yaml -f apps/base/ml-api/slo-rules.yaml 2>/dev/null
```

Expected: `wrote …` twice; `30` records per file (16 SLI + 14 metadata); `110 0` for both files (lines only added, so the SLI rules are unchanged); two `configured (server dry run)` lines.

- [ ] **Step 3: Update the SLO spec**

In `docs/superpowers/specs/2026-09-26-slo-sli-design.md`:

- Replace the Scope decision (lines 14–16) with:

```markdown
- **Scope:** SLI recording rules evaluated by vmalert, and Alertmanager
  routing. The metadata rules the SLO dashboards read came with the
  [dashboards spec](2026-09-26-grafana-dashboards-design.md). SLO targets,
  burn-rate alerts, notification channels, the SLO document and the error
  budget policy come later.
```

- Replace the "Targets later" decision (lines 31–35) with:

```markdown
- **Targets later:** Sloth requires an `objective` and an `alerting.name` on
  every SLO, even when it generates no alerts. Each SLO carries
  `objective: 99.9`, marked as a placeholder, and the alert name it will use
  later. The SLO dashboards measure burn rate and budget against the
  placeholder; no alert uses either yet. Latency thresholds are provisional.
```
- In the layout block replace `SLI rules only; writes VMRules` with `SLI and metadata rules; writes VMRules`.
- In the script description replace

```markdown
  -s '{"id":"sloth.dev/core/sli_rules/v1"}' < slo.yaml`. The validator rejects
```

with

```markdown
  -s '{"id":"sloth.dev/core/sli_rules/v1"}'
  -s '{"id":"sloth.dev/core/metadata_rules/v1"}' < slo.yaml`. The validator rejects
```

- After the paragraph ending `rule is the average of the 5-minute ratios over 4 weeks.` add:

```markdown

The metadata plugin adds 7 recording rules per SLO for Sloth's Grafana
dashboards: `slo:objective:ratio`, `slo:error_budget:ratio`,
`slo:time_period:days`, `slo:current_burn_rate:ratio` (5m window),
`slo:period_burn_rate:ratio` (4w), `slo:period_error_budget_remaining:ratio`
and `sloth_slo_info`.
```

- Replace the "When targets are chosen" paragraph with:

```markdown
When targets are chosen: set `objective` in each `slo.yaml`, and add
`sloth.dev/core/alert_rules/v1` to the plugin chain in the script. Every SLO
then gets the workbook's page and ticket alerts (Sloth's `google-28d` windows:
page 1 h/5 m and 6 h/30 m, ticket 1 d/2 h and 3 d/6 h), labelled
`sloth_severity=page|ticket`, which the Alertmanager routes below already
handle.
```
- Replace `into VMSingle: 4 SLOs × 8 windows = 32 recording rules, no alert rules.` with `into VMSingle: 4 SLOs × (8 SLI windows + 7 metadata rules) = 60 recording rules, no alert rules.`
- Replace the "Out of scope" text with:

```markdown
SLO targets, burn-rate alerts, notification channels, the SLO document and
error budget policy, CI.
```

- [ ] **Step 4: Commit, then run the drift check (spec criterion 2)**

```sh
git add scripts/slo-generate.sh apps/base/backend-api/slo-rules.yaml apps/base/ml-api/slo-rules.yaml docs/superpowers/specs/2026-09-26-slo-sli-design.md
git commit -m "feat: generate Sloth metadata rules for the SLO dashboards"
scripts/slo-generate.sh >/dev/null 2>&1 && git diff --exit-code -- 'apps/base/*/slo-rules.yaml'; echo "drift exit=$?"
```

Expected: `drift exit=0`.

---

## Task 2: Grafana with the SLO and component dashboards

**Files:**
- Modify: `infrastructure/base/monitoring/helmrelease.yaml` (`grafana`, `syncJob`)
- Modify: `infrastructure/base/monitoring/kustomization.yaml`
- Create: `infrastructure/base/monitoring/dashboards/kustomization.yaml`
- Create (downloaded): `infrastructure/base/monitoring/dashboards/slos/{sloth-overview,sloth-detail}.json`, `infrastructure/base/monitoring/dashboards/components/{flux-cluster,vm-single,node-exporter-full,k8s-pods,postgres}.json`
- Modify: `docs/superpowers/specs/2026-09-26-victoriametrics-metrics-collection-design.md`

- [ ] **Step 1: Switch on Grafana, switch off the sync job**

In `infrastructure/base/monitoring/helmrelease.yaml` replace

```yaml
    grafana:
      enabled: false
```

with

```yaml
    # Dashboards come from ConfigMaps in git (dashboards/); the sidecar puts each
    # into the folder its grafana_folder annotation names. No persistence: every
    # dashboard is provisioned, so a restart loses nothing. Nothing is downloaded
    # at startup: Grafana's preinstalled plugins (drilldown apps, extra data
    # sources) are off; the dashboards use only built-in panels and Prometheus.
    grafana:
      enabled: true
      grafana.ini:
        plugins:
          preinstall_disabled: true
      sidecar:
        dashboards:
          folderAnnotation: grafana_folder
          provider:
            foldersFromFilesStructure: true
```

and replace

```yaml
    defaultDashboards:
      enabled: false
```

with

```yaml
    defaultDashboards:
      enabled: false
    # Fetches dashboards and rules from the internet at deploy time; both are off.
    syncJob:
      enabled: false
```

- [ ] **Step 2: Download the pinned dashboards**

Run from the repo root (`python3 - <<'EOF' … EOF` or a throwaway file):

```python
# Throwaway: downloads the pinned dashboards. Run once from the repo root.
import os
import urllib.request

D = 'infrastructure/base/monitoring/dashboards'
SOURCES = {
    'slos/sloth-overview.json': 'https://grafana.com/api/dashboards/14643/revisions/2/download',
    'slos/sloth-detail.json': 'https://grafana.com/api/dashboards/14348/revisions/5/download',
    'components/flux-cluster.json': 'https://raw.githubusercontent.com/fluxcd/flux2-monitoring-example/7ab65dc8b90f7a6751d88f18bbb4e1bee33bf334/monitoring/configs/dashboards/cluster.json',
    'components/vm-single.json': 'https://raw.githubusercontent.com/VictoriaMetrics/VictoriaMetrics/v1.152.0/dashboards/victoriametrics.json',
    'components/node-exporter-full.json': 'https://grafana.com/api/dashboards/1860/revisions/45/download',
    'components/k8s-pods.json': 'https://raw.githubusercontent.com/dotdc/grafana-dashboards-kubernetes/v3.0.8/dashboards/k8s-views-pods.json',
    'components/postgres.json': 'https://raw.githubusercontent.com/prometheus-community/postgres_exporter/v0.20.1/postgres_mixin/dashboards/postgres-overview.json',
}
for path, url in SOURCES.items():
    os.makedirs(os.path.dirname(f'{D}/{path}'), exist_ok=True)
    with urllib.request.urlopen(urllib.request.Request(url, headers={'User-Agent': 'curl'}), timeout=60) as r:
        body = r.read()
    with open(f'{D}/{path}', 'wb') as f:
        f.write(body)
    print(path, len(body))
```

Expected: seven lines, sizes `19417`, `33394`, `34837`, `273911`, `468600`, `79545`, `35918`.

- [ ] **Step 3: Adapt the Sloth dashboards**

Run from the repo root:

```python
# Throwaway: adapts Sloth's grafana.com dashboards 14348 (rev 5) and 14643 (rev 2)
# for provisioning and the 28-day period. Run once from the repo root.
import json

D = 'infrastructure/base/monitoring/dashboards/slos'

def load(name):
    # Import inputs can't be resolved when provisioned; use the dashboards' own
    # data source variable instead.
    with open(f'{D}/{name}') as f:
        d = json.loads(f.read().replace('${DS_PROMETHEUS}', '${Datasource}'))
    del d['__inputs']
    return d

def save(name, d):
    with open(f'{D}/{name}', 'w') as f:
        f.write(json.dumps(d, indent=2, ensure_ascii=False) + '\n')

def panel(d, pid):
    return next(p for p in d['panels'] if p['id'] == pid)

d = load('sloth-overview.json')
panel(d, 110)['title'] = 'Budget remaining 28 day window'
save('sloth-overview.json', d)

d = load('sloth-detail.json')
# SLI window picker: our period rule is ratio_rate4w, not ratio_rate30d.
w = next(v for v in d['templating']['list'] if v['name'] == 'sli_window')
w['query'] = w['query'].replace('30d', '4w')
for o in w['options']:
    if o['value'] == '30d':
        o['text'] = o['value'] = '4w'
# The calendar-month budget is a second, different window: drop the stat ...
d['panels'] = [p for p in d['panels'] if p['id'] != 76]
p = panel(d, 12)
p['description'] = 'A rolling window of the total period (28d) error budget remaining.'
p['targets'][0]['legendFormat'] = 'Remaining error budget (28d window)'
# ... and turn the month burn chart into the rolling 28-day budget over time.
p = panel(d, 66)
p['title'] = 'Error budget remaining (rolling 28d)'
p['description'] = 'Error budget remaining over the rolling 28-day SLO period, over the last 28 days.'
p['timeFrom'] = '28d'
del p['timeShift']
p['targets'] = [{
    'datasource': {'type': 'prometheus', 'uid': '${Datasource}'},
    'expr': 'slo:period_error_budget_remaining:ratio{sloth_service="${service}", sloth_slo="${slo}"}',
    'legendFormat': 'Remaining error budget',
    'refId': 'A',
}]
p['fieldConfig']['overrides'] = p['fieldConfig']['overrides'][:1]  # drop the removed "Ideal constant consumption" line
save('sloth-detail.json', d)
```

Then check every file against the prototype:

```sh
cd infrastructure/base/monitoring/dashboards && shasum -a 256 -c - <<'EOF'
aa96099bc21038a4bb2ee0dcdccc01eef7e342b21db9a43ce93c1d7ce64b2beb  slos/sloth-detail.json
e5ebe04d5b5c547bd0b9b5017a8f4a41c15919b8ae3c64bc7b11a99c8c92028e  slos/sloth-overview.json
2a52d416ca7fee166c7703524332d3c8e586808c21ff5679a259c9dd744ed309  components/flux-cluster.json
e1282688933864c649c78a91352b434843b4e863c744128adc11d4d2b10b6bfd  components/k8s-pods.json
184c6b7409f306da75525d7772f71945b10cea23ad16b5d78c4698ea0ea51986  components/node-exporter-full.json
54906f590027076f7e918bca58269ff636df7886723a1f99caf4b283c0d9fe0d  components/postgres.json
7eb59c1e6887fd22aed5959b4dd32598729c420ac90c6dd950888ae27b759d80  components/vm-single.json
EOF
cd - >/dev/null
grep -c 'DS_PROMETHEUS' infrastructure/base/monitoring/dashboards/slos/*.json
```

Expected: seven `OK`; `0` for both Sloth files. The component files match their upstream download byte for byte; the Sloth files match the tested edits.

- [ ] **Step 4: Generate the ConfigMaps**

`infrastructure/base/monitoring/dashboards/kustomization.yaml`:

```yaml
# Grafana dashboards (docs/superpowers/specs/2026-09-26-grafana-dashboards-design.md).
# Each file becomes a ConfigMap; Grafana's sidecar loads every ConfigMap labelled
# grafana_dashboard=1 into the folder named by its grafana_folder annotation.
# Clusters replace one in their overlay with a configMapGenerator entry of the
# same name and `behavior: replace`.
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: monitoring
generatorOptions:
  disableNameSuffixHash: true # nothing mounts them by name; stable names keep overrides simple
  labels:
    grafana_dashboard: "1"
configMapGenerator:
  # Sloth, edited for provisioning and the 28-day period (see the spec):
  # https://grafana.com/grafana/dashboards/14643 revision 2
  - name: dashboard-sloth-overview
    files: [slos/sloth-overview.json]
    options: {annotations: {grafana_folder: SLOs}}
  # https://grafana.com/grafana/dashboards/14348 revision 5
  - name: dashboard-sloth-detail
    files: [slos/sloth-detail.json]
    options: {annotations: {grafana_folder: SLOs}}
  # Upstream, unedited. To update, download again at the new pin.
  # https://github.com/fluxcd/flux2-monitoring-example/blob/7ab65dc8b90f7a6751d88f18bbb4e1bee33bf334/monitoring/configs/dashboards/cluster.json
  - name: dashboard-flux-cluster
    files: [components/flux-cluster.json]
    options: {annotations: {grafana_folder: Components}}
  # https://github.com/VictoriaMetrics/VictoriaMetrics/blob/v1.152.0/dashboards/victoriametrics.json
  - name: dashboard-vm-single
    files: [components/vm-single.json]
    options: {annotations: {grafana_folder: Components}}
  # https://grafana.com/grafana/dashboards/1860 revision 45
  - name: dashboard-node-exporter-full
    files: [components/node-exporter-full.json]
    options: {annotations: {grafana_folder: Components}}
  # https://github.com/dotdc/grafana-dashboards-kubernetes/blob/v3.0.8/dashboards/k8s-views-pods.json
  - name: dashboard-k8s-pods
    files: [components/k8s-pods.json]
    options: {annotations: {grafana_folder: Components}}
  # https://github.com/prometheus-community/postgres_exporter/blob/v0.20.1/postgres_mixin/dashboards/postgres-overview.json
  - name: dashboard-postgres
    files: [components/postgres.json]
    options: {annotations: {grafana_folder: Components}}
```

In `infrastructure/base/monitoring/kustomization.yaml` add `  - dashboards` after `  - blackbox-exporter.yaml`.

- [ ] **Step 5: Render**

```sh
kubectl kustomize infrastructure/devops-cs > /tmp/infra.yaml
python3 - <<'EOF'
import yaml, collections
docs = [d for d in yaml.safe_load_all(open('/tmp/infra.yaml')) if d]
cms = [d for d in docs if d['kind'] == 'ConfigMap' and d['metadata']['name'].startswith('dashboard-')]
print(len(cms), dict(collections.Counter(c['metadata']['annotations']['grafana_folder'] for c in cms)),
      all(c['metadata']['namespace'] == 'monitoring' and c['metadata']['labels'] == {'grafana_dashboard': '1'} for c in cms))
EOF
python3 -c 'import yaml; d=[x for x in yaml.safe_load_all(open("/tmp/infra.yaml")) if x and x["kind"]=="HelmRelease" and x["metadata"]["name"]=="victoria-metrics-k8s-stack"][0]; print(yaml.safe_dump(d["spec"]["values"]))' > /tmp/values.yaml
docker run --rm -v /tmp/values.yaml:/values.yaml alpine/helm:3.17.3 template victoria-metrics-k8s-stack victoria-metrics-k8s-stack \
  --repo https://victoriametrics.github.io/helm-charts/ --version 0.93.0 -n monitoring -f /values.yaml > /tmp/chart.yaml
grep -E 'sync-job|preinstall_disabled|FOLDER_ANNOTATION|foldersFromFilesStructure|image: .*grafana/grafana' /tmp/chart.yaml
rm /tmp/infra.yaml /tmp/values.yaml /tmp/chart.yaml
```

Expected: `7 {'SLOs': 2, 'Components': 5} True`; then `foldersFromFilesStructure: true`, `preinstall_disabled = true`, `- name: FOLDER_ANNOTATION`, `image: "docker.io/grafana/grafana:13.1.1"`, and no `sync-job` line. A render error means a values typo.

- [ ] **Step 6: Update the metrics-collection spec**

In `docs/superpowers/specs/2026-09-26-victoriametrics-metrics-collection-design.md`:

- Components table: replace `| Grafana, default rules, default dashboards | off | Out of scope |` with

```markdown
| Grafana | on | Dashboards provisioned from git, no persistence, nothing downloaded at startup ([dashboards spec](2026-09-26-grafana-dashboards-design.md)) |
| Default rules, default dashboards, dashboard sync job | off | Rules and dashboards come from this repo |
```

- Out of scope: replace `Grafana, alerting, SLOs, logging, ingress, high availability.` with `Dashboards, alerting and SLOs (each has its own spec), logging, ingress, high availability.`

- [ ] **Step 7: Commit**

```sh
git add infrastructure/base/monitoring docs/superpowers/specs/2026-09-26-victoriametrics-metrics-collection-design.md
git commit -m "feat: Grafana with provisioned SLO and component dashboards"
```

---

## Task 3: Apps and Platform boards

**Files:**
- Create: `infrastructure/base/monitoring/dashboards/overview/apps.json`, `infrastructure/base/monitoring/dashboards/overview/platform.json`
- Modify: `infrastructure/base/monitoring/dashboards/kustomization.yaml`
- Modify: `TEMP-NOTES.md`

- [ ] **Step 1: Apps board**

`infrastructure/base/monitoring/dashboards/overview/apps.json`:

```json
{
  "uid": "apps",
  "title": "Apps",
  "description": "Golden signals of ml-api and backend-api for user traffic (POST /predict, POST /process), and postgres. SLO view: SLOs folder.",
  "tags": ["overview"],
  "time": {"from": "now-6h", "to": "now"},
  "refresh": "30s",
  "schemaVersion": 41,
  "templating": {"list": [{"name": "datasource", "label": "Data source", "type": "datasource", "query": "prometheus"}]},
  "panels": [
    {"id": 1, "type": "row", "title": "ml-api: POST /predict", "collapsed": false, "gridPos": {"x": 0, "y": 0, "w": 24, "h": 1}, "panels": []},
    {"id": 2, "type": "timeseries", "title": "Traffic", "gridPos": {"x": 0, "y": 1, "w": 6, "h": 8},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "reqps"}, "overrides": []},
     "targets": [{"refId": "A", "legendFormat": "{{status}}", "expr": "sum by (status) (rate(ml_api_requests_total{endpoint=\"/predict\"}[$__rate_interval]))"}]},
    {"id": 3, "type": "timeseries", "title": "Errors: failed predict probes", "gridPos": {"x": 6, "y": 1, "w": 6, "h": 8},
     "description": "Share of failed synthetic POST /predict probes. The ml-api server only ever records status 200.",
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "percentunit", "min": 0}, "overrides": []},
     "links": [{"title": "SLO detail", "url": "/d/slo-detail?var-service=ml-api&var-slo=predict-availability"}],
     "targets": [{"refId": "A", "legendFormat": "failed", "expr": "1 - avg_over_time(probe_success{job=\"probe/ml-api/predict\"}[$__rate_interval])"}]},
    {"id": 4, "type": "timeseries", "title": "Latency", "gridPos": {"x": 12, "y": 1, "w": 6, "h": 8},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "s", "min": 0}, "overrides": []},
     "links": [{"title": "SLO detail", "url": "/d/slo-detail?var-service=ml-api&var-slo=predict-latency"}],
     "targets": [
       {"refId": "A", "legendFormat": "p50", "expr": "histogram_quantile(0.5, sum by (le) (rate(ml_api_request_duration_seconds_bucket{endpoint=\"/predict\"}[$__rate_interval])))"},
       {"refId": "B", "legendFormat": "p90", "expr": "histogram_quantile(0.9, sum by (le) (rate(ml_api_request_duration_seconds_bucket{endpoint=\"/predict\"}[$__rate_interval])))"},
       {"refId": "C", "legendFormat": "p99", "expr": "histogram_quantile(0.99, sum by (le) (rate(ml_api_request_duration_seconds_bucket{endpoint=\"/predict\"}[$__rate_interval])))"}]},
    {"id": 5, "type": "stat", "title": "Pods ready / desired", "gridPos": {"x": 18, "y": 1, "w": 6, "h": 4},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"color": {"mode": "fixed", "fixedColor": "text"}}, "overrides": []},
     "links": [{"title": "Pods", "url": "/d/k8s_views_pods?var-namespace=ml-api"}],
     "targets": [
       {"refId": "A", "legendFormat": "ready", "expr": "max(kube_deployment_status_replicas_available{namespace=\"ml-api\",deployment=\"ml-api\"})"},
       {"refId": "B", "legendFormat": "desired", "expr": "max(kube_deployment_spec_replicas{namespace=\"ml-api\",deployment=\"ml-api\"})"}]},
    {"id": 6, "type": "stat", "title": "Container restarts", "gridPos": {"x": 18, "y": 5, "w": 6, "h": 4},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"color": {"mode": "fixed", "fixedColor": "text"}, "decimals": 0}, "overrides": []},
     "links": [{"title": "Pods", "url": "/d/k8s_views_pods?var-namespace=ml-api"}],
     "targets": [{"refId": "A", "instant": true, "expr": "sum(increase(kube_pod_container_status_restarts_total{namespace=\"ml-api\"}[$__range]))"}]},
    {"id": 7, "type": "timeseries", "title": "CPU vs limit", "gridPos": {"x": 0, "y": 9, "w": 8, "h": 7},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "percentunit", "min": 0}, "overrides": []},
     "links": [{"title": "Pods", "url": "/d/k8s_views_pods?var-namespace=ml-api"}],
     "targets": [{"refId": "A", "legendFormat": "{{pod}}", "expr": "sum by (pod) (rate(container_cpu_usage_seconds_total{namespace=\"ml-api\",container=\"ml-api\"}[$__rate_interval])) / sum by (pod) (kube_pod_container_resource_limits{namespace=\"ml-api\",container=\"ml-api\",resource=\"cpu\"})"}]},
    {"id": 8, "type": "timeseries", "title": "Memory vs limit", "gridPos": {"x": 8, "y": 9, "w": 8, "h": 7},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "percentunit", "min": 0}, "overrides": []},
     "links": [{"title": "Pods", "url": "/d/k8s_views_pods?var-namespace=ml-api"}],
     "targets": [{"refId": "A", "legendFormat": "{{pod}}", "expr": "sum by (pod) (container_memory_working_set_bytes{namespace=\"ml-api\",container=\"ml-api\"}) / sum by (pod) (kube_pod_container_resource_limits{namespace=\"ml-api\",container=\"ml-api\",resource=\"memory\"})"}]},
    {"id": 9, "type": "timeseries", "title": "CPU throttling", "gridPos": {"x": 16, "y": 9, "w": 8, "h": 7},
     "description": "Share of CPU periods in which the container was throttled. Explains latency spikes while average CPU looks fine.",
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "percentunit", "min": 0}, "overrides": []},
     "links": [{"title": "Pods", "url": "/d/k8s_views_pods?var-namespace=ml-api"}],
     "targets": [{"refId": "A", "legendFormat": "{{pod}}", "expr": "sum by (pod) (rate(container_cpu_cfs_throttled_periods_total{namespace=\"ml-api\",container=\"ml-api\"}[$__rate_interval])) / sum by (pod) (rate(container_cpu_cfs_periods_total{namespace=\"ml-api\",container=\"ml-api\"}[$__rate_interval]))"}]},

    {"id": 10, "type": "row", "title": "backend-api: POST /process", "collapsed": false, "gridPos": {"x": 0, "y": 16, "w": 24, "h": 1}, "panels": []},
    {"id": 11, "type": "timeseries", "title": "Traffic", "gridPos": {"x": 0, "y": 17, "w": 5, "h": 8},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "reqps"}, "overrides": []},
     "targets": [{"refId": "A", "legendFormat": "{{status}}", "expr": "sum by (status) (rate(backend_api_requests_total{endpoint=\"/process\"}[$__rate_interval]))"}]},
    {"id": 12, "type": "timeseries", "title": "Errors: 5xx share", "gridPos": {"x": 5, "y": 17, "w": 5, "h": 8},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "percentunit", "min": 0}, "overrides": []},
     "links": [{"title": "SLO detail", "url": "/d/slo-detail?var-service=backend-api&var-slo=process-availability"}],
     "targets": [{"refId": "A", "legendFormat": "5xx", "expr": "(sum(rate(backend_api_requests_total{endpoint=\"/process\",status=~\"5..\"}[$__rate_interval])) or vector(0)) / sum(rate(backend_api_requests_total{endpoint=\"/process\"}[$__rate_interval]))"}]},
    {"id": 13, "type": "timeseries", "title": "DB queries by status", "gridPos": {"x": 10, "y": 17, "w": 5, "h": 8},
     "description": "pool_exhausted: the pod's connection pool was empty and /process returned 503.",
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "ops"}, "overrides": []},
     "targets": [{"refId": "A", "legendFormat": "{{status}}", "expr": "sum by (status) (rate(backend_api_db_queries_total[$__rate_interval]))"}]},
    {"id": 14, "type": "timeseries", "title": "Latency", "gridPos": {"x": 15, "y": 17, "w": 5, "h": 8},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "s", "min": 0}, "overrides": []},
     "links": [{"title": "SLO detail", "url": "/d/slo-detail?var-service=backend-api&var-slo=process-latency"}],
     "targets": [
       {"refId": "A", "legendFormat": "p50", "expr": "histogram_quantile(0.5, sum by (le) (rate(backend_api_request_duration_seconds_bucket{endpoint=\"/process\"}[$__rate_interval])))"},
       {"refId": "B", "legendFormat": "p90", "expr": "histogram_quantile(0.9, sum by (le) (rate(backend_api_request_duration_seconds_bucket{endpoint=\"/process\"}[$__rate_interval])))"},
       {"refId": "C", "legendFormat": "p99", "expr": "histogram_quantile(0.99, sum by (le) (rate(backend_api_request_duration_seconds_bucket{endpoint=\"/process\"}[$__rate_interval])))"}]},
    {"id": 15, "type": "stat", "title": "Pods ready / desired", "gridPos": {"x": 20, "y": 17, "w": 4, "h": 4},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"color": {"mode": "fixed", "fixedColor": "text"}}, "overrides": []},
     "links": [{"title": "Pods", "url": "/d/k8s_views_pods?var-namespace=backend-api"}],
     "targets": [
       {"refId": "A", "legendFormat": "ready", "expr": "max(kube_deployment_status_replicas_available{namespace=\"backend-api\",deployment=\"backend-api\"})"},
       {"refId": "B", "legendFormat": "desired", "expr": "max(kube_deployment_spec_replicas{namespace=\"backend-api\",deployment=\"backend-api\"})"}]},
    {"id": 16, "type": "stat", "title": "Container restarts", "gridPos": {"x": 20, "y": 21, "w": 4, "h": 4},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"color": {"mode": "fixed", "fixedColor": "text"}, "decimals": 0}, "overrides": []},
     "links": [{"title": "Pods", "url": "/d/k8s_views_pods?var-namespace=backend-api"}],
     "targets": [{"refId": "A", "instant": true, "expr": "sum(increase(kube_pod_container_status_restarts_total{namespace=\"backend-api\"}[$__range]))"}]},
    {"id": 17, "type": "timeseries", "title": "CPU vs limit", "gridPos": {"x": 0, "y": 25, "w": 6, "h": 7},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "percentunit", "min": 0}, "overrides": []},
     "links": [{"title": "Pods", "url": "/d/k8s_views_pods?var-namespace=backend-api"}],
     "targets": [{"refId": "A", "legendFormat": "{{pod}}", "expr": "sum by (pod) (rate(container_cpu_usage_seconds_total{namespace=\"backend-api\",container=\"backend-api\"}[$__rate_interval])) / sum by (pod) (kube_pod_container_resource_limits{namespace=\"backend-api\",container=\"backend-api\",resource=\"cpu\"})"}]},
    {"id": 18, "type": "timeseries", "title": "Memory vs limit", "gridPos": {"x": 6, "y": 25, "w": 6, "h": 7},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "percentunit", "min": 0}, "overrides": []},
     "links": [{"title": "Pods", "url": "/d/k8s_views_pods?var-namespace=backend-api"}],
     "targets": [{"refId": "A", "legendFormat": "{{pod}}", "expr": "sum by (pod) (container_memory_working_set_bytes{namespace=\"backend-api\",container=\"backend-api\"}) / sum by (pod) (kube_pod_container_resource_limits{namespace=\"backend-api\",container=\"backend-api\",resource=\"memory\"})"}]},
    {"id": 19, "type": "timeseries", "title": "CPU throttling", "gridPos": {"x": 12, "y": 25, "w": 6, "h": 7},
     "description": "Share of CPU periods in which the container was throttled. Explains latency spikes while average CPU looks fine.",
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "percentunit", "min": 0}, "overrides": []},
     "links": [{"title": "Pods", "url": "/d/k8s_views_pods?var-namespace=backend-api"}],
     "targets": [{"refId": "A", "legendFormat": "{{pod}}", "expr": "sum by (pod) (rate(container_cpu_cfs_throttled_periods_total{namespace=\"backend-api\",container=\"backend-api\"}[$__rate_interval])) / sum by (pod) (rate(container_cpu_cfs_periods_total{namespace=\"backend-api\",container=\"backend-api\"}[$__rate_interval]))"}]},
    {"id": 20, "type": "timeseries", "title": "DB connections in use", "gridPos": {"x": 18, "y": 25, "w": 6, "h": 7},
     "description": "Per pod. Pool max per pod: 10 (DB_POOL_MAX default).",
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"min": 0, "decimals": 0}, "overrides": []},
     "targets": [{"refId": "A", "legendFormat": "{{pod}}", "expr": "sum by (pod) (backend_api_db_connections_active)"}]},

    {"id": 21, "type": "row", "title": "postgres: backend-api's database", "collapsed": false, "gridPos": {"x": 0, "y": 32, "w": 24, "h": 1}, "panels": []},
    {"id": 22, "type": "stat", "title": "Up", "gridPos": {"x": 0, "y": 33, "w": 4, "h": 4},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"color": {"mode": "thresholds"}, "thresholds": {"mode": "absolute", "steps": [{"color": "red", "value": null}, {"color": "green", "value": 1}]}}, "overrides": []},
     "links": [{"title": "Postgres", "url": "/d/wGgaPlciz"}],
     "targets": [{"refId": "A", "legendFormat": "pg_up", "expr": "pg_up"}]},
    {"id": 23, "type": "stat", "title": "Connections used", "gridPos": {"x": 4, "y": 33, "w": 4, "h": 4},
     "description": "Server connections in use, of max_connections.",
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "percentunit", "color": {"mode": "fixed", "fixedColor": "text"}}, "overrides": []},
     "links": [{"title": "Postgres", "url": "/d/wGgaPlciz"}],
     "targets": [{"refId": "A", "expr": "sum(pg_stat_database_numbackends) / max(pg_settings_max_connections)"}]}
  ]
}
```

- [ ] **Step 2: Platform board**

`infrastructure/base/monitoring/dashboards/overview/platform.json`:

```json
{
  "uid": "platform",
  "title": "Platform",
  "description": "Is GitOps converging, are workloads healthy, is the node healthy, is monitoring healthy. In the order you'd debug.",
  "tags": ["overview"],
  "time": {"from": "now-6h", "to": "now"},
  "refresh": "30s",
  "schemaVersion": 41,
  "templating": {"list": [{"name": "datasource", "label": "Data source", "type": "datasource", "query": "prometheus"}]},
  "panels": [
    {"id": 1, "type": "row", "title": "GitOps (Flux)", "collapsed": false, "gridPos": {"x": 0, "y": 0, "w": 24, "h": 1}, "panels": []},
    {"id": 2, "type": "stat", "title": "Flux objects not Ready", "gridPos": {"x": 0, "y": 1, "w": 4, "h": 11},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"color": {"mode": "thresholds"}, "thresholds": {"mode": "absolute", "steps": [{"color": "green", "value": null}, {"color": "red", "value": 1}]}}, "overrides": []},
     "links": [{"title": "Flux Cluster Stats", "url": "/d/flux-cluster"}],
     "targets": [{"refId": "A", "expr": "count(gotk_resource_info{ready!=\"True\"}) or vector(0)"}]},
    {"id": 3, "type": "table", "title": "Flux objects", "gridPos": {"x": 4, "y": 1, "w": 20, "h": 11},
     "description": "revision: the commit or chart version each object has applied.",
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {}, "overrides": []},
     "links": [{"title": "Flux Cluster Stats", "url": "/d/flux-cluster"}],
     "transformations": [
       {"id": "filterFieldsByName", "options": {"include": {"names": ["customresource_kind", "exported_namespace", "name", "ready", "suspended", "revision"]}}},
       {"id": "organize", "options": {"renameByName": {"customresource_kind": "kind", "exported_namespace": "namespace"}}}],
     "targets": [{"refId": "A", "instant": true, "range": false, "format": "table", "expr": "gotk_resource_info"}]},

    {"id": 4, "type": "row", "title": "Workloads", "collapsed": false, "gridPos": {"x": 0, "y": 12, "w": 24, "h": 1}, "panels": []},
    {"id": 5, "type": "stat", "title": "Pods not ready", "gridPos": {"x": 0, "y": 13, "w": 8, "h": 6},
     "description": "Per namespace. Completed (Succeeded) pods don't count.",
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"color": {"mode": "thresholds"}, "thresholds": {"mode": "absolute", "steps": [{"color": "green", "value": null}, {"color": "red", "value": 1}]}}, "overrides": []},
     "options": {"textMode": "value_and_name"},
     "links": [{"title": "Pods", "url": "/d/k8s_views_pods"}],
     "targets": [{"refId": "A", "legendFormat": "{{namespace}}", "expr": "sum by (namespace) (kube_pod_status_ready{condition=\"false\"} * on(namespace, pod) group_left() (1 - kube_pod_status_phase{phase=\"Succeeded\"}))"}]},
    {"id": 6, "type": "table", "title": "Containers restarted in the time range", "gridPos": {"x": 8, "y": 13, "w": 8, "h": 6},
     "description": "Empty while healthy.",
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"decimals": 0}, "overrides": []},
     "links": [{"title": "Pods", "url": "/d/k8s_views_pods"}],
     "transformations": [
       {"id": "filterFieldsByName", "options": {"include": {"names": ["namespace", "pod", "container", "Value"]}}},
       {"id": "organize", "options": {"renameByName": {"Value": "restarts"}}}],
     "targets": [{"refId": "A", "instant": true, "range": false, "format": "table", "expr": "sum by (namespace, pod, container) (increase(kube_pod_container_status_restarts_total[$__range])) > 0"}]},
    {"id": 7, "type": "table", "title": "Deployments with unavailable replicas", "gridPos": {"x": 16, "y": 13, "w": 8, "h": 6},
     "description": "Empty while healthy.",
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"decimals": 0}, "overrides": []},
     "transformations": [
       {"id": "filterFieldsByName", "options": {"include": {"names": ["namespace", "deployment", "Value"]}}},
       {"id": "organize", "options": {"renameByName": {"Value": "unavailable"}}}],
     "targets": [{"refId": "A", "instant": true, "range": false, "format": "table", "expr": "sum by (namespace, deployment) (kube_deployment_status_replicas_unavailable) > 0"}]},

    {"id": 8, "type": "row", "title": "Node", "collapsed": false, "gridPos": {"x": 0, "y": 19, "w": 24, "h": 1}, "panels": []},
    {"id": 9, "type": "stat", "title": "Node Ready", "gridPos": {"x": 0, "y": 20, "w": 4, "h": 6},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"color": {"mode": "thresholds"}, "thresholds": {"mode": "absolute", "steps": [{"color": "red", "value": null}, {"color": "green", "value": 1}]}}, "overrides": []},
     "options": {"textMode": "value_and_name"},
     "links": [{"title": "Node Exporter Full", "url": "/d/rYdddlPWk"}],
     "targets": [{"refId": "A", "legendFormat": "{{node}}", "expr": "max by (node) (kube_node_status_condition{condition=\"Ready\",status=\"true\"})"}]},
    {"id": 10, "type": "timeseries", "title": "CPU used", "gridPos": {"x": 4, "y": 20, "w": 7, "h": 6},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "percentunit", "min": 0, "max": 1}, "overrides": []},
     "links": [{"title": "Node Exporter Full", "url": "/d/rYdddlPWk"}],
     "targets": [{"refId": "A", "legendFormat": "cpu", "expr": "1 - avg(rate(node_cpu_seconds_total{mode=\"idle\"}[$__rate_interval]))"}]},
    {"id": 11, "type": "timeseries", "title": "Memory used", "gridPos": {"x": 11, "y": 20, "w": 7, "h": 6},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "percentunit", "min": 0, "max": 1}, "overrides": []},
     "links": [{"title": "Node Exporter Full", "url": "/d/rYdddlPWk"}],
     "targets": [{"refId": "A", "legendFormat": "{{instance}}", "expr": "1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes"}]},
    {"id": 12, "type": "timeseries", "title": "Disk used", "gridPos": {"x": 18, "y": 20, "w": 6, "h": 6},
     "description": "The node's root disk; on k3d, the disk holding /var/lib/rancher/k3s.",
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "percentunit", "min": 0, "max": 1}, "overrides": []},
     "links": [{"title": "Node Exporter Full", "url": "/d/rYdddlPWk"}],
     "targets": [{"refId": "A", "legendFormat": "disk", "expr": "max(1 - node_filesystem_avail_bytes{mountpoint=~\"/|/var/lib/rancher/k3s\"} / node_filesystem_size_bytes{mountpoint=~\"/|/var/lib/rancher/k3s\"})"}]},

    {"id": 13, "type": "row", "title": "Monitoring", "collapsed": false, "gridPos": {"x": 0, "y": 26, "w": 24, "h": 1}, "panels": []},
    {"id": 14, "type": "table", "title": "Scrape targets down", "gridPos": {"x": 0, "y": 27, "w": 10, "h": 6},
     "description": "Empty while healthy. Why a target is down: kubectl -n monitoring port-forward svc/vmagent-victoria-metrics-k8s-stack 8429, then http://localhost:8429/targets",
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"color": {"mode": "fixed", "fixedColor": "red"}, "custom": {"cellOptions": {"type": "color-background"}}}, "overrides": []},
     "transformations": [{"id": "filterFieldsByName", "options": {"include": {"names": ["job", "instance"]}}}],
     "targets": [{"refId": "A", "instant": true, "range": false, "format": "table", "expr": "up == 0"}]},
    {"id": 15, "type": "stat", "title": "VMSingle data size", "gridPos": {"x": 10, "y": 27, "w": 5, "h": 6},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "bytes", "color": {"mode": "fixed", "fixedColor": "text"}}, "overrides": []},
     "links": [{"title": "VictoriaMetrics - single-node", "url": "/d/wNf0q_kZk"}],
     "targets": [{"refId": "A", "expr": "sum(vm_data_size_bytes)"}]},
    {"id": 16, "type": "stat", "title": "VMSingle free disk", "gridPos": {"x": 15, "y": 27, "w": 5, "h": 6},
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"unit": "bytes", "color": {"mode": "fixed", "fixedColor": "text"}}, "overrides": []},
     "links": [{"title": "VictoriaMetrics - single-node", "url": "/d/wNf0q_kZk"}],
     "targets": [{"refId": "A", "expr": "min(vm_free_disk_space_bytes)"}]},
    {"id": 17, "type": "stat", "title": "vmalert rule errors", "gridPos": {"x": 20, "y": 27, "w": 4, "h": 6},
     "description": "Failed rule evaluations in the time range. Which rule and why: kubectl -n monitoring port-forward svc/vmalert-victoria-metrics-k8s-stack 8080, then http://localhost:8080",
     "datasource": {"type": "prometheus", "uid": "${datasource}"},
     "fieldConfig": {"defaults": {"decimals": 0, "color": {"mode": "thresholds"}, "thresholds": {"mode": "absolute", "steps": [{"color": "green", "value": null}, {"color": "red", "value": 1}]}}, "overrides": []},
     "targets": [{"refId": "A", "instant": true, "expr": "sum(increase(vmalert_recording_rules_errors_total[$__range])) + (sum(increase(vmalert_alerting_rules_errors_total[$__range])) or vector(0))"}]}
  ]
}
```

- [ ] **Step 3: Add them to the generator**

In `infrastructure/base/monitoring/dashboards/kustomization.yaml` insert after `configMapGenerator:`:

```yaml
  # Ours.
  - name: dashboard-apps
    files: [overview/apps.json]
    options: {annotations: {grafana_folder: Overview}}
  - name: dashboard-platform
    files: [overview/platform.json]
    options: {annotations: {grafana_folder: Overview}}
```

- [ ] **Step 4: Check the boards and render**

```sh
python3 - <<'EOF'
import json
for f in ('apps', 'platform'):
    d = json.load(open(f'infrastructure/base/monitoring/dashboards/overview/{f}.json'))
    ids = [p['id'] for p in d['panels']]
    cells = {}
    for p in d['panels']:
        g = p['gridPos']
        assert g['x'] + g['w'] <= 24, (f, p['id'])
        for x in range(g['x'], g['x'] + g['w']):
            for y in range(g['y'], g['y'] + g['h']):
                assert (x, y) not in cells, (f, p['id'], cells[(x, y)])
                cells[(x, y)] = p['id']
    print(f, d['uid'], len(ids) == len(set(ids)), sum(p['type'] != 'row' for p in d['panels']))
EOF
kubectl kustomize infrastructure/devops-cs > /tmp/infra.yaml
python3 - <<'EOF'
import yaml, collections
docs = [d for d in yaml.safe_load_all(open('/tmp/infra.yaml')) if d]
cms = [d for d in docs if d['kind'] == 'ConfigMap' and d['metadata']['name'].startswith('dashboard-')]
print(len(cms), dict(collections.Counter(c['metadata']['annotations']['grafana_folder'] for c in cms)),
      all(c['metadata']['namespace'] == 'monitoring' and c['metadata']['labels'] == {'grafana_dashboard': '1'} for c in cms))
EOF
rm /tmp/infra.yaml
```

Expected: `apps apps True 20`, `platform platform True 13` (valid JSON, unique panel ids, no overlapping panels); `9 {'Overview': 2, 'SLOs': 2, 'Components': 5} True`.

- [ ] **Step 5: Notes**

In `TEMP-NOTES.md`, "Where things live" table, add after the `.sourceignore` row:

```markdown
| Add or change a dashboard | `infrastructure/base/monitoring/dashboards/`: the JSON file plus one `configMapGenerator` entry in its `kustomization.yaml` |
| Use a different dashboard on one cluster | `infrastructure/<cluster>/monitoring/kustomization.yaml`: a `configMapGenerator` entry with the dashboard's name, `namespace: monitoring`, `behavior: replace` and a file with the same name |
```

- [ ] **Step 6: Commit**

```sh
git add infrastructure/base/monitoring/dashboards TEMP-NOTES.md
git commit -m "feat: Apps and Platform dashboards"
```

---

## Task 4: Verify on the cluster

**Files:** none changed.

- [ ] **Step 1: Push (user approval required)**

Ask the user to approve the push. After the OK:

```sh
git push
flux reconcile source git flux-system
```

- [ ] **Step 2: Ready, rules, no downloads (spec criteria 2 and 3)**

```sh
flux get kustomizations
flux get helmreleases -A
kubectl -n monitoring rollout status deploy/victoria-metrics-k8s-stack-grafana
kubectl -n monitoring get cm -l grafana_dashboard=1 --no-headers | wc -l
kubectl -n monitoring get jobs
kubectl get --raw "/api/v1/namespaces/monitoring/services/vmalert-victoria-metrics-k8s-stack:8080/proxy/api/v1/rules" \
  | jq -r '[.data.groups[].rules[]] | "rules=\(length) errors=\([.[] | select((.lastError // "") != "")] | length)"'
kubectl -n monitoring logs deploy/victoria-metrics-k8s-stack-grafana -c grafana | grep -c 'Installing plugin'
```

Expected: every Kustomization and both HelmReleases `True`; the Grafana rollout completes; `9`; no sync-job Job; `rules=60 errors=0`; `0`.

- [ ] **Step 3: Folders (spec criterion 4)**

```sh
kubectl -n monitoring port-forward svc/victoria-metrics-k8s-stack-grafana 3000:80 &
kubectl -n monitoring get secret victoria-metrics-k8s-stack-grafana -o jsonpath='{.data.admin-password}' | base64 -d; echo
```

Open http://localhost:3000 in a browser and log in as `admin` with that password. Expected under Dashboards: `Overview` (Apps, Platform), `SLOs` (High level Sloth SLOs, SLO / Detail), `Components` (Flux Cluster Stats, Kubernetes / Views / Pods, Node Exporter Full, Postgres Overview, VictoriaMetrics - single-node), nothing in `General`.

- [ ] **Step 4: Panels (spec criterion 4)**

Wait 5 minutes after Step 2 so the metadata rules have values. Open each of the 9 dashboards at its default time range.

Expected: no panel shows an error icon anywhere. On Apps, Platform, High level Sloth SLOs and SLO / Detail every panel shows data, except the problem lists, which are empty while healthy:
- Platform: "Containers restarted in the time range" (unless something restarted), "Deployments with unavailable replicas", "Scrape targets down";
- Sloth: the panels filtered to burn rate above 1x ("All burning rate (Filtered >1x)", "Exceeded burning rate SLOs", "Current exceeded burning rate SLOs") and "Burn rate (speed) magnitude" (burn rate above 0);
- Sloth warning/critical alert panels, which show `0`/`OK`.

Each stat shows one value per series it names ("Pods ready / desired": `ready` and `desired`; "VMSingle free disk": one value). On SLO / Detail pick `ml-api` / `predict-latency`: "Error budget remaining (rolling 28d)" draws a line, and the SLI window `4w` draws the SLI.

- [ ] **Step 5: Pods drill-down without a `cluster` label (spec criterion 5)**

Open http://localhost:3000/d/k8s_views_pods?var-namespace=ml-api.

Expected: the `cluster` picker is empty, and "Resources by container" and the CPU and memory panels show the `ml-api` container.

- [ ] **Step 6: Report**

Stop the port-forward (`kill %1`). Report Steps 2–5 to the user. No commit: this task changes no files.
