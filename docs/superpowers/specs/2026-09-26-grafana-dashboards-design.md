# Grafana dashboards

## Goal

Dashboards for four purposes: choosing SLO targets, watching app health,
watching the platform, and showing the case study. The SLO dashboards follow
the Google SRE workbook
([Implementing SLOs](https://sre.google/workbook/implementing-slos/): SLI
against the objective, error budget left, burn rate). The app and platform
dashboards show only what someone on call needs; everything else lives in
upstream drill-down dashboards.

## Decisions

- **Grafana:** from the existing `victoria-metrics-k8s-stack` HelmRelease
  (`grafana.enabled: true`, Grafana 13.1.1), with the chart's data sources:
  `VictoriaMetrics` (Prometheus type, uid `VictoriaMetrics`, default) and
  `Alertmanager`. No VictoriaMetrics Grafana plugin: Grafana would download it
  from the internet at every start.
- **Stateless:** no persistent volume (the chart's default `emptyDir`). Every
  dashboard is provisioned from git and can't be saved from the UI, so a
  restart loses nothing.
- **No runtime downloads:** `defaultDashboards.enabled: false` (already set)
  and `syncJob.enabled: false`. The sync job currently runs after every
  upgrade and syncs nothing.
- **Delivery:** dashboard JSON files in git, turned into ConfigMaps by
  Kustomize's `configMapGenerator`, loaded by the chart's Grafana sidecar.
  Rejected: grafana.com IDs in chart values (downloads at runtime, not
  reviewable), the Grafana operator (a second operator for a few files).
- **SLO dashboards:** Sloth's own two dashboards, not home-made ones; they
  need Sloth's metadata rules, which the script now generates.
- **App and platform:** two focused boards we own, plus pinned upstream
  drill-downs for depth.
- **Per cluster:** dashboards live in `base`; a cluster overlay can replace
  or remove one.
- **Access:** no ingress. `kubectl -n monitoring port-forward
  svc/victoria-metrics-k8s-stack-grafana 3000:80`, user `admin`, password in
  Secret `victoria-metrics-k8s-stack-grafana` key `admin-password` (generated
  by the chart).

## Layout

```
infrastructure/base/monitoring/
  kustomization.yaml        resources gain: dashboards
  helmrelease.yaml          grafana on (sidecar folders), syncJob off
  dashboards/
    kustomization.yaml      namespace monitoring; one configMapGenerator entry per file
    overview/   apps.json, platform.json                           ours
    slos/       sloth-overview.json, sloth-detail.json             Sloth, pinned, edited
    components/ flux-cluster.json, flux-control-plane.json,
                vm-single.json, vmagent.json, vmalert.json,
                node-exporter-full.json, k8s-pods.json,
                postgres.json                                      upstream, pinned, unedited
```

`dashboards/kustomization.yaml`:

- `namespace: monitoring`.
- `generatorOptions`: label `grafana_dashboard: "1"`, and
  `disableNameSuffixHash: true`. Nothing mounts these ConfigMaps by name, so
  a hash suffix would buy nothing; stable names make cluster overrides simple.
- One entry per file, named `dashboard-<file name without .json>` (12
  ConfigMaps), with the annotation `grafana_folder` set to `Overview`, `SLOs`
  or `Components`.
- A comment per upstream file with its source URL and pin.

HelmRelease values:

```yaml
grafana:
  enabled: true
  sidecar:
    dashboards:
      folderAnnotation: grafana_folder
      provider:
        foldersFromFilesStructure: true
syncJob:
  enabled: false
```

The sidecar (label `grafana_dashboard=1`, Grafana's namespace) writes each
ConfigMap into the folder its annotation names; Grafana creates the folders.

### Per-cluster override

In `infrastructure/<cluster>/monitoring/kustomization.yaml`:

```yaml
configMapGenerator:
  - name: dashboard-apps          # replace: same name, file with the same name
    namespace: monitoring
    behavior: replace
    files: [apps.json]
patches:
  - patch: |                      # remove
      $patch: delete
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: dashboard-node-exporter-full
        namespace: monitoring
```

Tested with Kustomize: the replacement keeps the base label and folder
annotation; the delete patch removes the ConfigMap.

## SLO dashboards (folder `SLOs`)

| File | Source | Shows |
|---|---|---|
| `sloth-overview.json` | grafana.com 14643, revision 2 | every SLO's burn rate, SLOs burning now, budget remaining |
| `sloth-detail.json` | grafana.com 14348, revision 5 | per SLO: SLI vs objective, current burn rate, budget remaining, burn-rate heatmap, page/ticket alert state |

**Rules they need:** `scripts/slo-generate.sh` adds
`sloth.dev/core/metadata_rules/v1` after `sli_rules`. Per SLO it adds 7
recording rules: `slo:objective:ratio`, `slo:error_budget:ratio`,
`slo:time_period:days` (28), `slo:current_burn_rate:ratio` (5m window),
`slo:period_burn_rate:ratio` (4w), `slo:period_error_budget_remaining:ratio`
and `sloth_slo_info`. vmalert then runs 4 × 15 = 60 recording rules. Burn
rate and budget are measured against the placeholder `objective: 99.9` until
real targets are set.

**Edits, made once and committed** (the source and revision stay in the
kustomization comment):

1. Data source: `${DS_PROMETHEUS}` becomes `${Datasource}`, the dashboards'
   own data source variable, and the `__inputs` import block is deleted. A
   provisioned dashboard can't resolve import inputs.
2. 28-day window: the Detail `sli_window` option `30d` becomes `4w`, the name
   of our period rule. Texts that say "30d" or "30 day" say "28d" / "28 day".
3. Calendar month: Detail's "month" remaining-budget stat is removed, and its
   "Month error budget burn chart" becomes "Error budget remaining (rolling
   28d)" on `slo:period_error_budget_remaining:ratio`, with a 28d panel time
   range. A calendar month is a different window from our 4-week rolling one
   and would show a second, different budget number.

Unchanged: the page/ticket panels read `ALERTS` and show 0 until burn-rate
alerts exist. All panel types are built into Grafana.

## Our boards (folder `Overview`)

Hand-written JSON, fixed UIDs `apps` and `platform`, a `${datasource}`
variable (type Prometheus), default range 6h, refresh 30s. `$__rate_interval`
in rates. Red only where the bad value is clear (not Ready > 0, targets down
> 0, `pg_up` = 0); CPU, memory and disk get no invented warning levels, those
come with the platform alerts. Panels link to the drill-down they summarize by
UID (`/d/<uid>`).

### Apps

One row per API with the SRE book's four golden signals, user traffic only
(`/predict`, `/process`; health, readiness and `/metrics` excluded like the
SLIs). Shown for ml-api; backend-api is the same with its names unless noted.

```
# Traffic, req/s by status
sum by (status) (rate(ml_api_requests_total{endpoint="/predict"}[$__rate_interval]))

# Errors
#   ml-api: failed predict probes (the server only ever records 200)
1 - avg_over_time(probe_success{job="probe/ml-api/predict"}[$__rate_interval])
#   backend-api: 5xx share, and DB queries by status (success, pool_exhausted, error)
(sum(rate(backend_api_requests_total{endpoint="/process",status=~"5.."}[$__rate_interval])) or vector(0))
  / sum(rate(backend_api_requests_total{endpoint="/process"}[$__rate_interval]))
sum by (status) (rate(backend_api_db_queries_total[$__rate_interval]))

# Latency, p50 / p90 / p99
histogram_quantile(0.99, sum by (le) (rate(ml_api_request_duration_seconds_bucket{endpoint="/predict"}[$__rate_interval])))

# Saturation, per pod: CPU and memory vs limit, CPU throttling
sum by (pod) (rate(container_cpu_usage_seconds_total{namespace="ml-api",container="ml-api"}[$__rate_interval]))
  / sum by (pod) (kube_pod_container_resource_limits{namespace="ml-api",container="ml-api",resource="cpu"})
sum by (pod) (container_memory_working_set_bytes{namespace="ml-api",container="ml-api"})
  / sum by (pod) (kube_pod_container_resource_limits{namespace="ml-api",container="ml-api",resource="memory"})
sum by (pod) (rate(container_cpu_cfs_throttled_periods_total{namespace="ml-api",container="ml-api"}[$__rate_interval]))
  / sum by (pod) (rate(container_cpu_cfs_periods_total{namespace="ml-api",container="ml-api"}[$__rate_interval]))
#   backend-api only: DB connections in use per pod (pool max 10, DB_POOL_MAX default)
sum by (pod) (backend_api_db_connections_active)

# Pods: ready vs desired, restarts in the range
kube_deployment_status_replicas_available{namespace="ml-api",deployment="ml-api"}
kube_deployment_spec_replicas{namespace="ml-api",deployment="ml-api"}
sum(increase(kube_pod_container_status_restarts_total{namespace="ml-api"}[$__range]))
```

Postgres row (backend-api's only dependency), links to the Postgres drill-down:

```
pg_up
sum(pg_stat_database_numbackends) / max(pg_settings_max_connections)
sum(rate(pg_stat_database_xact_commit[$__rate_interval]))
sum(increase(pg_stat_database_deadlocks[$__range]))
```

`ml_api_memory_bytes` is not used: it always reports 0. Memory comes from
cAdvisor.

### Platform

Four rows, in the order you'd debug:

```
# 1. GitOps (link: Flux Cluster Stats)
count(gotk_resource_info{ready!="True"}) or vector(0)
gotk_resource_info   # table: customresource_kind, exported_namespace, name, ready, suspended, revision

# 2. Workloads (link: Kubernetes / Views / Pods)
sum by (namespace) (kube_pod_status_ready{condition="false"}
  * on(namespace, pod) group_left() (1 - kube_pod_status_phase{phase="Succeeded"}))
sum by (namespace, pod, container) (increase(kube_pod_container_status_restarts_total[$__range])) > 0
sum by (namespace, deployment) (kube_deployment_status_replicas_unavailable) > 0
count(kube_persistentvolumeclaim_status_phase{phase!="Bound"} == 1) or vector(0)

# 3. Node (link: Node Exporter Full)
kube_node_status_condition{condition="Ready",status="true"}
1 - avg(rate(node_cpu_seconds_total{mode="idle"}[$__rate_interval]))
1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes
max(1 - node_filesystem_avail_bytes{mountpoint=~"/|/var/lib/rancher/k3s"}
      / node_filesystem_size_bytes{mountpoint=~"/|/var/lib/rancher/k3s"})

# 4. Monitoring itself (links: VictoriaMetrics single-node, vmagent, vmalert)
up == 0              # table: job, instance
sum(rate(vm_rows_inserted_total[$__rate_interval]))
sum(vm_cache_entries{type="storage/hour_metric_ids"})
sum(vm_data_size_bytes)
vm_free_disk_space_bytes
sum(increase(vmalert_recording_rules_errors_total[$__range]))
  + (sum(increase(vmalert_alerting_rules_errors_total[$__range])) or vector(0))
sum(increase(vmalert_iteration_missed_total[$__range]))
```

The disk query covers both a normal node (`/`) and k3d, where the node's `/`
is an overlay the chart excludes and k3s data sits on `/var/lib/rancher/k3s`.
The restart, unavailable-deployment and targets-down tables are problem lists:
empty while healthy.

All queries above returned data (or an empty problem list) on VMSingle on
2026-09-26 with `$__rate_interval` = 5m and `$__range` = 6h.

## Drill-downs (folder `Components`)

Committed exactly as downloaded. Updating one means downloading it again at
the new pin.

| File | Source, pin | UID |
|---|---|---|
| `flux-cluster.json` | [fluxcd/flux2-monitoring-example](https://github.com/fluxcd/flux2-monitoring-example) `monitoring/configs/dashboards/cluster.json` @ `7ab65dc8b90f` | `flux-cluster` |
| `flux-control-plane.json` | same repo, `control-plane.json` @ `7ab65dc8b90f` | `flux-control-plane` |
| `vm-single.json` | [VictoriaMetrics](https://github.com/VictoriaMetrics/VictoriaMetrics) `dashboards/victoriametrics.json` @ `v1.152.0` | `wNf0q_kZk` |
| `vmagent.json` | same repo, `dashboards/vmagent.json` @ `v1.152.0` | `G7Z9GzMGz` |
| `vmalert.json` | same repo, `dashboards/vmalert.json` @ `v1.152.0` | `LzldHAVnz` |
| `node-exporter-full.json` | grafana.com 1860, revision 45 | `rYdddlPWk` |
| `k8s-pods.json` | [dotdc/grafana-dashboards-kubernetes](https://github.com/dotdc/grafana-dashboards-kubernetes) `dashboards/k8s-views-pods.json` @ `v3.0.8` | `k8s_views_pods` |
| `postgres.json` | [postgres_exporter](https://github.com/prometheus-community/postgres_exporter) `postgres_mixin/dashboards/postgres-overview.json` @ `v0.20.1` | `wGgaPlciz` |

The pins match what runs: our `gotk_resource_info` config is copied from the
Flux example; VMSingle, vmagent and vmalert run v1.152.0; node-exporter
v1.11.1; the postgres-exporter sidecar v0.20.1 (chosen over grafana.com 9628,
which uses an old dashboard format).

Checked against VMSingle: every dashboard selects its data source through its
own Prometheus-type variable, so provisioning works unedited. The Flux, pod
and postgres dashboards use no metric we lack. Some panels stay empty because
the setup doesn't have what they show: node-exporter hardware panels (fans,
temperatures, CPU frequency) on a Docker VM, vmagent's Kafka and stream
aggregation panels, vmalert's alerting-rule panels until alerts exist. The
largest file is 469 KB, under the 1 MiB ConfigMap limit; Flux applies
server-side, so the 256 KB last-applied annotation limit doesn't apply.
`.sourceignore` already lets Flux fetch `infrastructure/`.

## Documentation updates

- Metrics-collection spec: Grafana moves from off to on; dashboard sync job
  off.
- SLO spec: the plugin chain includes `metadata_rules`, 60 recording rules;
  choosing targets later means setting `objective` and adding
  `sloth.dev/core/alert_rules/v1`. "Dashboards" leaves its scope and
  out-of-scope lines.
- `TEMP-NOTES.md`: where dashboards live and how to override one per cluster.

## Acceptance criteria

1. `kubectl kustomize infrastructure/devops-cs` renders 12 ConfigMaps named
   `dashboard-*` in `monitoring`, each with label `grafana_dashboard: "1"` and
   a `grafana_folder` annotation (2 `Overview`, 2 `SLOs`, 8 `Components`). The
   chart render has no sync-job Job.
2. The SLO drift check passes with the new plugin; vmalert reports 60 rules and
   0 errors.
3. `flux get kustomizations` and `flux get helmreleases -A` show all Ready;
   the Grafana pod is Ready.
4. The Grafana API lists 12 dashboards in the folders `Overview`, `SLOs`,
   `Components`, and `VictoriaMetrics` is the default data source.
5. A throwaway script runs every panel query of `apps`, `platform`,
   `sloth-overview` and `sloth-detail` through Grafana (`/api/ds/query`, with
   the dashboard variables filled in). Every query succeeds; every panel
   returns data except the problem lists (empty while healthy) and the
   page/ticket alert panels (0).
6. Each drill-down opens without errors, and one key panel has data: Flux
   Cluster Stats shows 9 objects; Kubernetes / Views / Pods shows the
   `ml-api` pods without a `cluster` label; Node Exporter Full, Postgres
   Overview and the three VictoriaMetrics dashboards show their up/uptime
   panels.
7. After deleting the Grafana pod, all 12 dashboards come back.

## Known limits

- The SLO dashboards measure against the 99.9 placeholder; their burn rates and
  budgets mean nothing until real targets are set.
- The 28-day numbers cover only the data collected so far until 28 days have
  passed.
- Dashboards can't be edited in the UI and saved; changes go through git.
  Exploring in the UI works; saving a copy is lost on restart.
- Upstream drill-downs are not updated automatically; each is re-downloaded
  at a new pin by hand.

## Out of scope

Platform alerts, SLO targets and burn-rate alerts, ingress and login other than
the admin user, Grafana persistence, dashboards for logs.
