# SLIs and SLO groundwork (Google SRE workbook)

## Goal

Measure the user-facing SLIs of ml-api and backend-api the way the Google SRE
workbook prescribes ([Implementing SLOs](https://sre.google/workbook/implementing-slos/),
[Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)), with the
recording windows and burn-rate alerts it describes. Since 2026-09-27 every SLO
has a target and Sloth generates its burn-rate alerts; routing and the
platform alerts are in the [alerting spec](2026-09-27-alerting-design.md).

## Decisions

- **Scope:** SLI recording rules evaluated by vmalert, SLO targets and their
  burn-rate alerts, and Alertmanager routing. The metadata rules the SLO
  dashboards read came with the
  [dashboards spec](2026-09-26-grafana-dashboards-design.md). Notification
  channels, the SLO document and the error budget policy come later.
- **SLI style:** the ratio of bad events to valid events, as the workbook
  recommends ("What to Measure: Using SLIs").
- **SLO window:** 30-day rolling window, Sloth's default and the period its
  Grafana dashboards are built for, so they run unedited. (Until 2026-09-27
  this was the workbook's 4 weeks, "Choosing an Appropriate Time Window",
  which needed edited dashboards.)
- **Generator:** [Sloth](https://github.com/slok/sloth) v0.16.0 as a
  Kubernetes controller. Each app ships a `PrometheusServiceLevel` object with
  its manifests; the controller turns it into a `PrometheusRule`, and the
  VictoriaMetrics operator converts that into a `VMRule` for vmalert. No
  generation step, no generated files in git, and it scales to many services
  pushed from a monorepo. (Until 2026-09-27 the Sloth CLI generated `VMRule`
  files that were committed; that needed a manual run and a drift check.)
  Rejected: hand-written rules (a home-made Sloth to maintain).
- **Common vs per app:** everything shared lives in one place
  (`infrastructure/base/monitoring/sloth.yaml`: Sloth version, window,
  validator, plugin chain; the blackbox probe module in
  `blackbox-exporter.yaml`). Each app only defines its SLIs in its own
  `slo.yaml`.
- **Targets (2026-09-27):** `objective: 99` for all four SLOs, a 30-day budget
  of 1% (7.2 h of full outage). ml-api's availability SLI has only 120 probes
  an hour, and at 99.9% two failed probes in an hour would page; at 99% a page
  needs about 9 minutes of full outage in an hour. The latency thresholds
  stay at 1 s and 0.25 s: measured p99 was 0.50 s and 0.01 s. Every SLO keeps
  its `alerting.name`, which Sloth requires.
- **Receivers:** `page` and `ticket` without integrations; alerts are visible
  in the Alertmanager UI.

## SLIs

Only user-facing requests count: `POST /predict` and `POST /process`, the
requests the load-generator sends. `/health`, `/ready` and `/metrics` are
probes and scrapes, not user requests, and are excluded. A 5xx response is bad;
every other response, including 4xx, is good (the workbook's HTTP rule). Each
service is measured across all its pods: `increase` per pod first, then sum.
MetricsQL's `rate` starts from a series' first sample, so it drops what a pod
counted before its first scrape. The backend creates its 5xx series only at
the first error, so `rate` hid a short 503 burst entirely in the failure test;
`increase` counts a new series' small first value. The error/total ratio
means the same either way.

| Service | SLO | Source | Bad events / valid events |
|---|---|---|---|
| ml-api | `predict-availability` | Black-box probe | failed `POST /predict` probes / all probes |
| ml-api | `predict-latency` | Server histogram | requests slower than 1 s (provisional) / all `/predict` |
| backend-api | `process-availability` | Server counter | `POST /process` with status 500 or 503 / all `POST /process` |
| backend-api | `process-latency` | Server histogram | requests slower than 0.25 s (provisional) / all `/process` |

Latency thresholds are today's p99 rounded up to the next histogram bucket
edge, with headroom (`/predict` p99 ≈ 0.50 s, `/process` p99 ≈ 0.10 s). The
`le` label values are exactly as the apps export them: `0.01`, `0.05`, `0.1`,
`0.25`, `0.5`, `1.0`, `2.5`, `5.0`, `10.0`. A query must use that exact text
(`le="1.0"`, not `le="1"`), or it matches nothing.

Queries (`{{.window}}` is filled in by Sloth):

```
# backend-api process-availability
error: sum(increase(backend_api_requests_total{endpoint="/process",status=~"5.."}[{{.window}}])) or vector(0)
total: sum(increase(backend_api_requests_total{endpoint="/process"}[{{.window}}]))

# backend-api process-latency
error: sum(increase(backend_api_request_duration_seconds_count{endpoint="/process"}[{{.window}}]))
       - sum(increase(backend_api_request_duration_seconds_bucket{endpoint="/process",le="0.25"}[{{.window}}]))
total: sum(increase(backend_api_request_duration_seconds_count{endpoint="/process"}[{{.window}}]))

# ml-api predict-availability
error: sum(count_over_time(probe_success{job="probe/ml-api/predict"}[{{.window}}]))
       - sum(sum_over_time(probe_success{job="probe/ml-api/predict"}[{{.window}}]))
total: sum(count_over_time(probe_success{job="probe/ml-api/predict"}[{{.window}}]))

# ml-api predict-latency
error: sum(increase(ml_api_request_duration_seconds_count{endpoint="/predict"}[{{.window}}]))
       - sum(increase(ml_api_request_duration_seconds_bucket{endpoint="/predict",le="1.0"}[{{.window}}]))
total: sum(increase(ml_api_request_duration_seconds_count{endpoint="/predict"}[{{.window}}]))
```

A status series only exists after its first occurrence, so the backend has no
5xx series while it is healthy. `or vector(0)` turns "no errors" into 0 instead
of no data; without it, Sloth's 30-day SLI would average only the 5-minute
windows that had errors.

### Why ml-api availability is black-box

The ml-api code (`/app/app.py`) only ever records `status="200"` for
`/predict`; the handler has no error path. Its real failure modes are invisible
to its own counters: memory growth until OOM kill (`MEM_ALLOC_MB`), `/health`
turning 503 after `HEALTH_TTL_SECONDS` and the liveness probe restarting the
pod, and refused connections during restarts.

A `VMProbe` named `predict` in the `ml-api` namespace sends an empty
`POST /predict` to `http://ml-api.ml-api.svc.cluster.local:8000/predict` every
30 s through the existing blackbox exporter; its job label is
`probe/ml-api/predict`. `/predict` reads no input and has no side effects, so
probing it is safe.
The workbook lists black-box monitoring as an SLI source and synthetic traffic
as a remedy for low-traffic services. The blackbox exporter's built-in
`http_2xx` module only sends GET, so its chart values gain a shared module:

```yaml
config:
  modules:
    http_post_2xx:
      prober: http
      http:
        method: POST
        preferred_ip_protocol: ip4
```

The backend is not probed: each `POST /process` inserts a row into
`documents`.

## Generation with the Sloth controller

```
infrastructure/base/monitoring/
  sloth.yaml               Sloth controller v0.16.0 (default 30d period, VM validator,
                           SLI and metadata rules) and the PrometheusRule CRD
  helmrelease.yaml         converter owner references on; waits for the CRD
  blackbox-exporter.yaml   adds the http_post_2xx module
apps/base/backend-api/
  slo.yaml                 PrometheusServiceLevel "backend-api"
apps/base/ml-api/
  slo.yaml                 PrometheusServiceLevel "ml-api"
  vmprobe.yaml             VMProbe "predict", the source of predict-availability
clusters/devops-cs/
  apps.yaml                health check for PrometheusServiceLevel
```

The chain from an SLO to vmalert:

1. Flux applies `slo.yaml` (a `PrometheusServiceLevel`) with the app.
2. The Sloth controller generates the rules and writes a `PrometheusRule` with
   the same name and namespace, owned by the `PrometheusServiceLevel`. Its
   plugin chain is set once, as controller arguments (a post-renderer in
   `sloth.yaml`, the chart has no values for it): `--disable-default-slo-plugins`,
   then `sloth.dev/contrib/validate_victoria_metrics/v1`, which rejects queries
   that are not valid MetricsQL, `sloth.dev/core/sli_rules/v1` and
   `sloth.dev/core/metadata_rules/v1`.
3. The VictoriaMetrics operator converts the `PrometheusRule` into a `VMRule`
   of the same name. `enable_converter_ownership` makes the `PrometheusRule` its
   owner; without it, deleting an SLO would leave its rules running. The
   operator only converts kinds whose CRD exists when it starts, so the stack's
   HelmRelease `dependsOn` the CRD release.

Only the `PrometheusRule` CRD of prometheus-operator is installed (chart
`prometheus-operator-crds`, every other CRD off). The Sloth chart's `commonPlugins`
(a git-sync sidecar pulling unpinned plugins from GitHub) and its `PodMonitor`
are off.

**Errors show in Flux.** Sloth's status has counters and a
`promOpRulesGenerated` flag but no conditions, and it sets `observedGeneration`
on failure too. Flux's default health check would call a failed SLO Ready. The
`apps` Kustomization therefore has `healthCheckExprs` for
`PrometheusServiceLevel`: in progress until Sloth has seen the current
generation, failed if `promOpRulesGenerated` is false, ready if it is true.
`kubectl get prometheusservicelevels -A` shows the same (`GEN OK`).

Sloth generates 8 recording rules per SLO, `slo:sli_error:ratio_rate{5m,30m,1h,2h,6h,1d,3d,30d}`,
labelled `sloth_id`, `sloth_service`, `sloth_slo` and `sloth_window`. These are
the windows the workbook's multiwindow, multi-burn-rate alerts need. The `30d`
rule is the average of the 5-minute ratios over 30 days. The rules are the
same as the CLI's; only `sloth_slo_info`'s `sloth_mode` and `sloth_spec` labels
differ.

Workflow: edit `slo.yaml` and commit. Flux applies apps after infrastructure,
so the `PrometheusServiceLevel`, `VMRule` and `VMProbe` CRDs exist first. With
services in a monorepo, its CI can check `slo.yaml` files before merge with
the same image: `sloth validate -i <dir> -n 'slo\.yaml$'` plus the plugin
arguments above.

Alerts: `sloth.dev/core/alert_rules/v1` in the plugin chain gives every SLO the
workbook's page and ticket alerts (Sloth's default `google-30d` windows: page
1 h/5 m at 14.4× and 6 h/30 m at 6×, ticket 1 d/2 h at 3× and 3 d/6 h at 1×),
labelled `sloth_severity=page|ticket`, without `for:`, as the workbook
advises. At 99% a page fires when both windows of a pair exceed 14.4% or 6%
errors.

SLOs are defined once in `base`; every cluster gets the same SLOs. A cluster
that needs another objective patches the `objective` field of the
`PrometheusServiceLevel` in its overlay.

## Runtime

Both components come from the existing `victoria-metrics-k8s-stack` chart,
switched on in `infrastructure/base/monitoring/helmrelease.yaml`:

- `vmalert.enabled: true`. It selects every `VMRule` in every namespace
  (`selectAllByDefault`), evaluates every 20 s and writes the recorded series
  into VMSingle: 4 SLOs × (8 SLI windows + 7 metadata rules) = 60 recording
  rules, plus 8 alert rules (page and ticket per SLO).
- `alertmanager.enabled: true`. The chart points vmalert at it. Routing (the
  full configuration is in the [alerting spec](2026-09-27-alerting-design.md)):
  `sloth_severity="page"` goes to receiver `page`, tickets to `ticket`, and a
  page silences the ticket of the same SLO (`equal: [sloth_id]`). That
  inhibit rule is the workbook's alert suppression: a fast burn also
  satisfies the slower conditions and would otherwise notify twice.

**Retention and storage:** VMSingle `retentionPeriod` goes from `14d` to the
chart's default `"1"`: one month, which VictoriaMetrics counts as 31 days (the
30-day window, and the longest calendar month for the Sloth dashboards' month
panels). Measured on 2026-09-26 with the formula
from VictoriaMetrics' sizing guide
(`sum(vm_data_size_bytes) / sum(vm_rows{type!~"indexdb.*"})`): 193 million
samples a day (2,234/s) at 2.21 bytes per sample, index included, before
background merges shrink the data. That is about 13 GB for 31 days, or 16 GB
with the 20% free space VictoriaMetrics recommends for merges. Old data is
deleted lazily, so usage can stay above that for a while.

- `base` drops its `5Gi` request, so every cluster gets the chart's default
  `20Gi`. On a storage class that enforces the size, 5Gi would fill in about
  12 days and VictoriaMetrics would stop accepting data.
- devops-cs keeps `5Gi` through a patch in
  `infrastructure/devops-cs/monitoring/kustomization.yaml`. `local-path`
  ignores the size and can't grow the existing volume, and a new volume would
  lose the stored metrics. The node disk has 312 GiB free.

Watch `vm_data_size_bytes`; if it grows too fast, drop the API server and etcd
histogram buckets first (the `ponytail:` note in the HelmRelease).

This updates the metrics-collection spec: vmalert and Alertmanager move from
off to on, retention from 14 days to 31 days (the chart default), and the base PVC from 5Gi to the
chart's 20Gi (devops-cs stays at 5Gi).

**Access:** no ingress. vmalert and Alertmanager UIs through
`kubectl port-forward`.

## Acceptance criteria

1. `flux get kustomizations` and `flux get helmreleases -A` show all Ready.
   `PrometheusServiceLevel` `backend-api` and `ml-api` show `GEN OK` true, their
   `VMRule`s `backend-api` and `ml-api`, `VMProbe` `predict` and the
   `VMAlertmanager` report operational. vmalert reports no rule evaluation
   errors. The target `probe/ml-api/predict` is up.
2. `slo:sli_error:ratio_rate5m` has exactly one series per SLO (4), and
   `slo:sli_error:ratio_rate30d` exists for all 4. While the apps are healthy,
   all values are about 0.
3. Each SLI detects its own failure (throwaway tests, Flux suspended, reverted
   afterwards):
   - ml-api scaled to 0: `predict-availability` rises above 0.
   - `CONN_RETURN_MODE=hold` on backend-api: `process-availability` rises
     above 0. Each pod keeps every connection, its pool (10) runs out and
     `/process` returns 503. `/ready` then fails too and the pods leave the
     Service; the 503s before that are enough. Postgres is untouched.
   - `RESPONSE_OVERHEAD_MS=1000` on ml-api: `predict-latency` rises above 0.
   - `QUERY_OVERHEAD_MS=300` on backend-api: `process-latency` rises above 0.
4. A `PrometheusServiceLevel` with invalid MetricsQL fails the `apps`
   Kustomization's health check; deleting a `PrometheusServiceLevel` deletes
   its `PrometheusRule` and `VMRule`.
5. VMSingle on devops-cs runs with the chart's `retentionPeriod: "1"` (31 days) and still requests
   `5Gi`.
6. Synthetic alerts sent to Alertmanager: a `sloth_severity="page"` alert
   goes to `page` and a ticket to `ticket`; a ticket with the same `sloth_id`
   as a firing page is inhibited, a ticket with another `sloth_id` is not.

## Known limits

- backend-api availability is measured by the server. Requests that never
  reach a pod (all pods down, refused connections) are not counted. The
  existing `GET /ready` probe stays a separate signal.
- ml-api availability is sampled: 2 probes a minute, so a blip shorter than
  30 s can be missed. It measures what the probe experiences; a problem that
  hits only real traffic stays hidden behind successful probes (the workbook's
  warning about synthetic traffic).
- Latency SLIs include failed requests; the histograms have no `status` label.
  ml-api's latency SLI also includes the probe requests, about 11% of
  `/predict` traffic.
- `backend_api_db_*` and `ml_api_memory_bytes` describe causes, not user
  experience; they are for diagnosis and dashboards, not SLIs.
- Sloth's 30-day SLI is the average of 5-minute ratios, not total bad events
  over total events. Close for steady traffic; bursty traffic skews it. It
  becomes meaningful 30 days after deployment.
- On devops-cs, disk use (about 16 GB estimated) exceeds the nominal 5Gi
  request; `local-path` doesn't enforce it.
- No notification leaves the cluster.
- The server-side queries rely on MetricsQL's `increase`; Prometheus'
  `increase` would drop a new series' first value again.
- The generated rules are not in git; `kubectl get vmrule -n <app> <app> -o yaml`
  shows them. A Sloth upgrade regenerates every SLO's rules in the cluster.

## Out of scope

Notification channels, the SLO document and error budget policy, CI.
