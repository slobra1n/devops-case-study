# SLIs and SLO groundwork (Google SRE workbook)

## Goal

Measure the user-facing SLIs of ml-api and backend-api the way the Google SRE
workbook prescribes ([Implementing SLOs](https://sre.google/workbook/implementing-slos/),
[Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)), and set up
the parts its alerting needs. Choosing targets later then only means filling in
numbers. This step records SLIs and routes alerts; it sets no targets and
creates no alert rules.

## Decisions

- **Scope:** SLI recording rules, vmalert and Alertmanager routing. Dashboards
  come next. SLO targets, burn-rate alerts, the SLO document and the error
  budget policy come later.
- **SLI style:** the ratio of bad events to valid events, as the workbook
  recommends ("What to Measure: Using SLIs").
- **SLO window:** 4-week rolling window, the workbook's general-purpose choice
  ("Choosing an Appropriate Time Window").
- **Generator:** [Sloth](https://github.com/slok/sloth) v0.16.0, run as a CLI
  from its pinned container image on a developer machine. Its output is
  committed as `VMRule` objects, so the cluster only runs VictoriaMetrics-native
  objects and reviewers see the exact rules in git. Rejected: Sloth's
  Kubernetes controller (needs Prometheus-operator CRDs, rules not visible in
  git) and hand-written rules (a home-made Sloth to maintain).
- **Common vs per app:** everything shared lives in one place
  (`scripts/slo-generate.sh`: Sloth version, window, validator, plugin chain;
  the blackbox probe module). Each app only defines its SLIs in its own
  `slo.yaml`.
- **Targets later:** Sloth requires an `objective` field, so every SLO carries
  `objective: 99.9` marked as a placeholder. Nothing uses it: the plugin chain
  generates SLI rules only. Latency thresholds are provisional.
- **Receivers:** `page` and `ticket` without integrations; alerts are visible in
  the Alertmanager UI.

## SLIs

Only user-facing requests count: `POST /predict` and `POST /process`, the
requests the load-generator sends. `/health`, `/ready` and `/metrics` are
probes and scrapes, not user requests, and are excluded. A 5xx response is bad;
every other response, including 4xx, is good (the workbook's HTTP rule). Each
service is measured across all its pods: rate per pod first, then sum.

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
error: sum(rate(backend_api_requests_total{endpoint="/process",status=~"5.."}[{{.window}}])) or vector(0)
total: sum(rate(backend_api_requests_total{endpoint="/process"}[{{.window}}]))

# backend-api process-latency
error: sum(rate(backend_api_request_duration_seconds_count{endpoint="/process"}[{{.window}}]))
       - sum(rate(backend_api_request_duration_seconds_bucket{endpoint="/process",le="0.25"}[{{.window}}]))
total: sum(rate(backend_api_request_duration_seconds_count{endpoint="/process"}[{{.window}}]))

# ml-api predict-availability
error: sum(count_over_time(probe_success{job="probe/ml-api/predict"}[{{.window}}]))
       - sum(sum_over_time(probe_success{job="probe/ml-api/predict"}[{{.window}}]))
total: sum(count_over_time(probe_success{job="probe/ml-api/predict"}[{{.window}}]))

# ml-api predict-latency
error: sum(rate(ml_api_request_duration_seconds_count{endpoint="/predict"}[{{.window}}]))
       - sum(rate(ml_api_request_duration_seconds_bucket{endpoint="/predict",le="1.0"}[{{.window}}]))
total: sum(rate(ml_api_request_duration_seconds_count{endpoint="/predict"}[{{.window}}]))
```

A status series only exists after its first occurrence, so the backend has no
5xx series while it is healthy. `or vector(0)` turns "no errors" into 0 instead
of no data; without it, Sloth's 4-week SLI would average only the 5-minute
windows that had errors.

### Why ml-api availability is black-box

The ml-api code (`/app/app.py`) only ever records `status="200"` for
`/predict`; the handler has no error path. Its real failure modes are invisible
to its own counters: memory growth until OOM kill (`MEM_ALLOC_MB`), `/health`
turning 503 after `HEALTH_TTL_SECONDS` and the liveness probe restarting the
pod, and refused connections during restarts. The backend code does record its
failures: 503 when the connection pool is exhausted, 500 on any other
exception.

A `VMProbe` named `predict` in the `ml-api` namespace sends `POST /predict`
with body `{}` to `http://ml-api.ml-api.svc.cluster.local:8000/predict` every
30 s through the existing blackbox exporter; its job label is
`probe/ml-api/predict`. `/predict` has no side effects, so probing it is safe.
The workbook lists black-box monitoring as an SLI source and synthetic traffic
as a remedy for low-traffic services. The blackbox exporter's built-in
`http_2xx` module only sends GET, so its chart values gain a shared module:

```yaml
config:
  modules:
    http_post_2xx:
      prober: http
      timeout: 5s
      http:
        method: POST
        headers:
          Content-Type: application/json
        body: "{}"
        preferred_ip_protocol: ip4
```

The backend is not probed: each `POST /process` inserts a row into
`documents`.

## Generation with Sloth

```
scripts/
  slo-generate.sh          common: Sloth image v0.16.0, 28d windows, VM validator,
                           SLI rules only; writes VMRules; --check mode
infrastructure/base/monitoring/
  blackbox-exporter.yaml   adds the http_post_2xx module
apps/base/backend-api/
  slo.yaml                 Sloth spec (not a Kubernetes object, not in kustomization.yaml)
  slo-rules.yaml           generated VMRule "backend-api-slo", do not edit; in kustomization.yaml
apps/base/ml-api/
  slo.yaml, slo-rules.yaml (VMRule "ml-api-slo")
  vmprobe.yaml             VMProbe "predict", the source of predict-availability
```

`scripts/slo-generate.sh`:

- Finds every `apps/base/*/slo.yaml` and runs, for each:
  `docker run --rm ghcr.io/slok/sloth:v0.16.0 generate --default-slo-period=28d
  --disable-default-slo-plugins -s '{"id":"sloth.dev/contrib/validate_victoria_metrics/v1"}'
  -s '{"id":"sloth.dev/core/sli_rules/v1"}'`. The validator rejects queries
  that are not valid MetricsQL.
- Wraps Sloth's rule groups into a `VMRule` named `<folder>-slo` in the
  namespace named like the folder (both apps use their folder name as
  namespace), and writes it to `slo-rules.yaml` with a "generated, do not
  edit" header.
- `--check` generates into a temporary directory and fails if any committed
  `slo-rules.yaml` differs. There is no CI; run it before committing.

Sloth generates 8 recording rules per SLO, `slo:sli_error:ratio_rate{5m,30m,1h,2h,6h,1d,3d,4w}`,
labelled `sloth_id`, `sloth_service`, `sloth_slo` and `sloth_window`. These are
the windows the workbook's multiwindow, multi-burn-rate alerts need. The `4w`
rule is the average of the 5-minute ratios over 4 weeks.

Workflow: edit `slo.yaml`, run the script, commit both files. Flux applies
apps after infrastructure, so the `VMRule` and `VMProbe` CRDs exist first.

When targets are chosen: set `objective` in each `slo.yaml`, and add
`sloth.dev/core/metadata_rules/v1` and `sloth.dev/core/alert_rules/v1` to the
plugin chain in the script. Every SLO then gets the workbook's page and ticket
alerts (Sloth's `google-28d` windows: page 1 h/5 m and 6 h/30 m, ticket
1 d/2 h and 3 d/6 h), labelled `sloth_severity=page|ticket`.

SLOs are defined once in `base`; every cluster gets the same SLOs. Per-cluster
SLO overrides are not built.

## Runtime

Both components come from the existing `victoria-metrics-k8s-stack` chart,
switched on in `infrastructure/base/monitoring/helmrelease.yaml`:

- `vmalert.enabled: true`. It selects every `VMRule` in every namespace
  (`selectAllByDefault`), evaluates every 20 s and writes the recorded series
  into VMSingle: 4 SLOs × 8 windows = 32 recording rules, no alert rules.
- `alertmanager.enabled: true`, with this configuration:

```yaml
route:
  receiver: ticket                                  # unrouted alerts still reach a human
  group_by: [alertname, sloth_id]
  routes:
    - matchers: ['sloth_severity="page"']
      receiver: page
    - matchers: ['sloth_severity="ticket"']
      receiver: ticket
receivers:
  - name: page
  - name: ticket
inhibit_rules:                                      # a page silences the ticket of the same SLO
  - source_matchers: ['sloth_severity="page"']
    target_matchers: ['sloth_severity="ticket"']
    equal: [sloth_id]
```

The inhibit rule is the workbook's alert suppression: a fast burn also
satisfies the slower conditions and would otherwise notify twice.

**Retention:** VMSingle `retentionPeriod` goes from `14d` to `30d` (4-week
window plus 2 days margin). At today's ingest (about 274 million samples a day,
1–4 bytes each) that is roughly 8–33 GB. The PVC request stays `5Gi`:
`local-path` does not enforce it and cannot expand volumes, and changing it
would mean recreating the volume and losing stored metrics. The node disk has
312 GiB free. Real clusters set the size in their overlay. Watch
`vm_data_size_bytes`; if it grows too fast, drop the API server and etcd
histogram buckets first (the `ponytail:` note in the HelmRelease).

This updates the metrics-collection spec: vmalert and Alertmanager move from
off to on, retention from 14 days to 30 days.

**Access:** no ingress. vmalert and Alertmanager UIs through
`kubectl port-forward`.

## Acceptance criteria

1. `flux get kustomizations` and `flux get helmreleases -A` show all Ready.
   `VMRule` `backend-api-slo` and `ml-api-slo` and `VMProbe` `predict` report
   operational. vmalert reports no rule evaluation errors. The target
   `probe/ml-api/predict` is up.
2. `slo:sli_error:ratio_rate5m` has exactly one series per SLO (4), and
   `slo:sli_error:ratio_rate4w` exists for all 4. While the apps are healthy,
   all values are about 0.
3. Each SLI detects its own failure (throwaway tests, Flux suspended, reverted
   afterwards):
   - ml-api scaled to 0: `predict-availability` rises above 0.
   - postgres scaled to 0: `process-availability` rises above 0. Afterwards
     restart backend-api (known workaround in `TEMP-NOTES.md`).
   - `RESPONSE_OVERHEAD_MS=1000` on ml-api: `predict-latency` rises above 0.
   - `QUERY_OVERHEAD_MS=300` on backend-api: `process-latency` rises above 0.
4. Synthetic alerts posted to Alertmanager's API: a `sloth_severity="page"`
   alert is routed to `page`, a `ticket` alert to `ticket`; a ticket with the
   same `sloth_id` as a firing page is inhibited, a ticket with another
   `sloth_id` is not.
5. `scripts/slo-generate.sh --check` passes; it fails after editing a
   `slo.yaml` without regenerating, and passes again after regenerating.
6. VMSingle runs with `retentionPeriod: 30d`.

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
- Sloth's 4-week SLI is the average of 5-minute ratios, not total bad events
  over total events. Close for steady traffic; bursty traffic skews it. It
  becomes meaningful 28 days after deployment.
- Disk use (8–33 GB estimated) exceeds the nominal 5Gi request.
- No notification leaves the cluster.
- `--check` only runs when someone runs it; there is no CI.

## Out of scope

SLO targets, burn-rate alerts, notification channels, dashboards, the SLO
document and error budget policy, CI.
