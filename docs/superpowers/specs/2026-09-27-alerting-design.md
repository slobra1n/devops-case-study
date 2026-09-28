# Alerting

> Working note from building this with AI agents; not canonical. The
> manifests and the [README](../../../README.md) are, and they win where
> this file disagrees with them.

## Goal

Alert on what users feel (SLO burn rates) and on what breaks the platform
(nodes, pods, kubelet, scrape targets, VictoriaMetrics, postgres, Flux), with
published rules plus five of our own for the gaps they leave, routed to a
`page` and a `ticket` receiver. No notification leaves the cluster yet.

## Decisions

- **SLO alerts:** Sloth's `alert_rules` plugin, targets 99%
  ([SLO spec](2026-09-26-slo-sli-design.md)). 8 alerts, a page and a ticket
  per SLO, labelled `sloth_severity`.
- **Platform alerts:** the chart's default rules, not hand-written ones. The
  chart's sync job (`syncJob.enabled: true`, a Helm post-install/upgrade hook)
  downloads the rule files and creates one `VMRule` per group in `monitoring`,
  deleting groups it no longer generates. Sources, pinned to what runs here
  (the chart's defaults follow `master`/`main`):

| Source | Pin | Covers |
|---|---|---|
| kube-prometheus `alertmanager`, `kubernetesControlPlane`, `kubePrometheus`, `kubeStateMetrics`, `nodeExporter` rules | `v0.19.0` | Alertmanager, kubelet, pods/deployments, resources, storage, node, TargetDown, Watchdog |
| VictoriaMetrics `alerts-health`, `alerts-vmagent`, `alerts-vmalert`, `alerts-single-node` | `v1.152.0` | VictoriaMetrics components, disk running out |
| VictoriaMetrics operator `vmoperator-rules` | `v0.74.1` | operator errors |
| monitoring-mixins `postgres-exporter/alerts.yaml` (extra source) | commit `50c46fcb` | postgres down, connections, deadlocks |
| VictoriaLogs `alerts-health`, `alerts-vlagent`, `alerts-vlogs` | `v1.52.0` | VictoriaLogs and VLAgent ([logging spec](2026-09-27-logging-design.md)) |

  The chart already skips the groups of components k3s doesn't expose (API
  server, scheduler, controller-manager, etcd scrapes are off). Adaptations,
  all as chart values:
  - `kubernetes-system`: `labelRewrites` `job="apiserver"` → `job="kubelet"`,
    so `KubeClientErrors` reads the API server metrics k3s serves on the
    kubelet's `/metrics`.
  - `kube-apiserver-availability.rules` off: recording rules only, for the
    API server SLO alerts that are skipped with the API server scrape. They
    select `job="apiserver"` and recorded nothing.
  - `PostgresHasTooManyRollbacks` off: every backend `/ready` check (`SELECT 1`)
    ends in a rollback when the pool takes the connection back, 1,620 an hour,
    exactly the `/ready` rate; not an application error.
  - `NodeClockNotSynchronising` off on devops-cs only (overlay patch): the
    Docker Desktop VM's kernel clock has no NTP daemon and always reports
    unsynchronised, though it matches the host within milliseconds.
    `NodeClockSkewDetected` still watches the offset.
  - `GroupIterationReset` off on devops-cs only (same patch, 2026-09-28): the
    VM's wall clock steps back by up to 0.4 ms a few times a minute, measured
    against the monotonic clock in a pod, and vmalert resets a group's
    schedule on each backward step. It fired for 34 groups on a fresh cluster
    with no missed iterations and 30 ms evaluations.
    `TooManyMissedIterations` still catches slow evaluations.

- **Our own rules** (since 2026-09-27): the SLOs count requests inside the
  apps. A pod that fails readiness leaves its Service, gets no requests, and
  its counters stop, so a full outage burns no error budget; the published pod
  alerts catch it only as tickets after 15 minutes. Five rules close that gap
  and name the causes the case study's apps can produce:

| Rule | File | Severity | Fires when |
|---|---|---|---|
| `DeploymentUnavailable` | `extraRules` in `infrastructure/base/monitoring/helmrelease.yaml` (VMRule `victoria-metrics-k8s-stack-workload-alerts`) | critical (page) | a Deployment outside `kube-system`, `flux-system` and `monitoring` has had no available pod for 1 minute |
| `ContainerOOMKilled` | same | warning | a container restarted in the last 10 minutes and its last termination was `OOMKilled` |
| `ContainerMemoryNearLimit` | same | warning | a container's working set has been above 90% of its memory limit for 5 minutes |
| `BackendDbPoolNearlyFull` | `apps/base/backend-api/alerts.yaml` | warning | a backend-api pod has held 8 or more of its 10 pool connections for 1 minute |
| `BackendDbQueryErrors` | same | warning | any backend-api query ended `pool_exhausted` or `error` in the last 5 minutes |

  Each has a `dashboard` link: the Namespace (Pods) or Pod board, or the apps
  board for the backend rules.

- **Flux:** Flux's own mechanism, not a PromQL rule: a notification-controller
  `Provider` of type `alertmanager` and an `Alert` for error events of every
  GitRepository and Kustomization in `flux-system` and every HelmRepository
  and HelmRelease in `monitoring` (`infrastructure/base/monitoring/flux-alerts.yaml`).
  Labels: `alertname=Flux<Kind><Reason>`, `severity=error`. Events have no end
  time, so Alertmanager's `global.resolve_timeout` is 1 h, as Flux's docs
  suggest; vmalert's alerts carry end times and are unaffected.
- **Routing:** kube-prometheus' published Alertmanager config plus the Sloth
  routes:

| Alerts | Receiver |
|---|---|
| `alertname="Watchdog"` (always firing) | `watchdog` |
| `alertname="InfoInhibitor"` | `null` |
| `sloth_severity="page"` | `page` |
| `severity="critical"` | `page` |
| everything else: Sloth tickets, `warning`, `info`, Flux `error` | `ticket` |

  Inhibition: a Sloth page silences the ticket of the same `sloth_id`;
  kube-prometheus' three rules (critical silences warning/info of the same
  alert and namespace, warning silences info, `InfoInhibitor` silences info).
  Grouped by `namespace`, `alertname`, `sloth_id`; `repeat_interval` 12 h.
- **Receivers:** `page`, `ticket`, `watchdog`, `null`, all without
  integrations. A channel is later one integration block per receiver;
  `watchdog` then goes to a dead man's switch.

## Acceptance criteria

1. All Flux Kustomizations and HelmReleases Ready; the sync-job Job completed.
2. vmalert: 34 default groups (179 alerts, 53 recording rules, including
   VictoriaLogs' 3 groups), the Sloth rules (60 recording, 8 alerts) and our
   2 groups (5 alerts), no rule errors.
3. While healthy, only `Watchdog` (→ `watchdog`) and `InfoInhibitor` (→
   `null`) fire; info-level alerts are silenced by `InfoInhibitor`.
4. `amtool config routes test`: Watchdog → `watchdog`, Sloth page → `page`,
   Sloth ticket → `ticket`, `severity=critical` → `page`, `warning` →
   `ticket`, Flux `error` → `ticket`.
5. A failing Flux Kustomization appears in Alertmanager as
   `FluxKustomization…` with `severity=error`.
6. Fault tests fire our rules: `CONN_RETURN_MODE=hold` on backend-api raises
   `BackendDbQueryErrors`, `BackendDbPoolNearlyFull` and then
   `DeploymentUnavailable` within 5 minutes, and no SLO alert; the ml-api
   image with `MEM_ALLOC_MB` in a scratch namespace raises
   `ContainerOOMKilled`, then `DeploymentUnavailable`, which stays firing
   while the pod crash-loops.

## Known limits

- No notification leaves the cluster; nobody is told when `Watchdog` stops.
- The default rules are downloaded from GitHub at every Helm upgrade. If
  GitHub is unreachable, the upgrade fails and is retried; while it fails,
  `infrastructure` is not Ready and new app deploys wait (apps depend on it).
- The rules live in the cluster, not in git; `kubectl -n monitoring get vmrule
  -l app.kubernetes.io/managed-by=sync-job` lists them.
- Alerts that can't fire here for lack of their metrics: Alertmanager cluster
  alerts (one replica), kubelet certificate alerts (not exposed by k3s),
  node RAID/systemd/bonding alerts (not in the Docker VM), kube-state-metrics'
  List/Watch error alerts (its v2.19.1 telemetry exports no list/watch
  counters; `TargetDown` covers it being down).
- `count:up0` records nothing while every target is up, so vmalert's
  `RecordingRulesNoData` (severity `info`) is pending for it; `InfoInhibitor`
  keeps info alerts from notifying, as kube-prometheus intends.
- Flux alerts are events: one alert per failure event, resolved after an hour
  unless the failure repeats.
- `DeploymentUnavailable` resolves 5 minutes after the app recovers
  (`keep_firing_for`), so a crash loop pages once.
- `BackendDbPoolNearlyFull` has the pool size (10, the app's default
  `DB_POOL_MAX`) written into its threshold; the app exports no pool maximum.
- `BackendDbQueryErrors` resolves 5 minutes after the last failed query. A
  leaked pool gets no more requests once its pods are unready, so from then
  on only `BackendDbPoolNearlyFull` and the page stay.

## Out of scope

Notification channels and a dead man's switch, secret management for them
(Secrets are plain text in git today), the SLO document, the error budget
policy.
