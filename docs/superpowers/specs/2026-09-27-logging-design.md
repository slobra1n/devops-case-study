# Logs with VictoriaLogs

## Goal

Every container's log in one place, searchable next to the metrics in
Grafana and in VictoriaLogs' own UI, from the same VictoriaMetrics stack.

## Decisions

- **Stack:** VictoriaLogs from the existing `victoria-metrics-k8s-stack`
  HelmRelease, managed by the VictoriaMetrics operator: `vlsingle.enabled` and
  `vlagent.enabled`. Both run v1.52.0, the version the chart pins. Rejected:
  Loki or a separate log agent (a second stack beside VictoriaMetrics).
  With VictoriaLogs on, the chart also puts an internal VMAuth
  (`vmauth-victoria-metrics-k8s-stack-internal`) in front of VMSingle and
  VLSingle and points vmalert's data source at it, so vmalert can evaluate
  MetricsQL and LogsQL rules. All rule evaluation now goes through that pod.
- **Collection:** VLAgent with its Kubernetes collector (the chart's default,
  `k8sCollector.enabled: true`) runs on every node, reads every container's
  log from the node and adds pod, namespace and container fields. It sends
  them to VLSingle. All containers: apps, postgres, Flux, monitoring,
  kube-system.
- **Storage:** chart defaults: retention one month (31 days, same as the
  metrics), a 20Gi volume. Measured volume on 2026-09-27: about 200 log lines a
  minute across all pods.
- **Viewing:**
  - Grafana: data source `VictoriaLogs` (uid `VictoriaLogs`), plugin
    `victoriametrics-logs-datasource` 0.32.0. Grafana installs it from
    grafana.com in the background after every start (`plugins.preinstall`).
    Tested with Grafana 13.1.1: without internet Grafana still starts and only
    this data source is missing until the next start. The chart's own
    VictoriaLogs data source is not used: the chart adds it only with its
    `plugins` value, which installs before start, and then Grafana doesn't
    start at all without internet (tested). The data source is therefore
    listed in `defaultDatasources.extra`.
    The apps log plain text such as `INFO:     ...`, with no level field, so
    the data source has log level rules (`jsonData.logLevelRules`) that take
    the level from that prefix. A `level` field, as in Flux's JSON logs, wins
    over the rules.
  - VictoriaLogs' built-in UI: `kubectl -n monitoring port-forward
    svc/vlsingle-victoria-metrics-k8s-stack 9428` and
    `http://localhost:9428/select/vmui`.
- **Alerts:** only the logging stack's own health, the chart's published
  rules pinned to v1.52.0 (`vlhealth`, `vlagent`, `victorialogs` in
  `defaultRules.sources`; [alerting spec](2026-09-27-alerting-design.md)). No
  alerts on log content.

## Acceptance criteria

1. HelmRelease and Kustomizations Ready; VLSingle and VLAgent operational; one
   VLAgent pod per node.
2. VictoriaLogs returns logs from `ml-api`, `backend-api`, `postgres`,
   `flux-system` and `monitoring`, with pod, namespace and container fields.
3. Grafana lists the `VictoriaLogs` data source, the plugin is installed and
   only that plugin; a query through the data source returns log lines, and
   ml-api and backend-api lines show their level (`INFO`), not `unknown`.
4. vmalert loads the VictoriaLogs health groups with no rule errors.

## Known limits

- Grafana's logs data source depends on grafana.com at every Grafana start.
- No log-based alerts; errors in logs are found by searching.
- Logs are kept one month; older logs are gone.
- Other plain-text formats (kube-system logfmt, operator console logs,
  postgres `LOG:`) show level `unknown`; add a rule when one matters.
