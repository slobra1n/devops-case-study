# Temporary notes: folder structure changes

## Goal

Make the apps wait for postgres. In Flux, `dependsOn` works only between Flux
Kustomizations. Postgres and the apps were applied by the same one (`apps`),
so there was no way to order them.

## Change 1: split `apps/` into `base/` and an overlay

- `apps/base/<app>/`: what each app is: objects, ports, probes, defaults.
  The same in every environment.
- `apps/devops-cs/<app>/`: what is specific to this environment. It uses the
  same folder layout as `base/`, so every app's environment-specific files sit
  in one place.
- Why: a later staging/production can reuse the base and change only what
  differs (image tags, replicas, credentials) without copying folders.
- YAGNI: with only one environment, the overlay changes little today. I see
  that argument, but I still chose this structure because I expect more
  environments in the future.

## Change 2: credentials moved to the overlay

- Base only refers to the `postgres-credentials` Secret by name.
- Each environment provides its own Secret (`secret.yaml` next to the app).
- Why: credentials differ per environment and should not be inherited from base.
- Not done on purpose: no secret management yet. The values are still
  plaintext test values.

## Change 3: postgres moved to its own `databases/` layer

- Postgres is meant to be an app, but all apps depend on it.
- To order it, postgres needs its own Flux Kustomization. If that Flux
  Kustomization pointed into `apps/`, the folder would say "app" while Flux
  treats it as a separate step. That would confuse maintainers.
- So the rule is: one top-level folder = one layer = one Flux Kustomization.
  Order: `infra-controllers` → `databases` → `apps`.
- Adding another database (e.g. mysql) later means adding a folder in
  `databases/`. `clusters/` does not change, and the apps wait for every
  database through the single `databases` layer.

## Change 4: one folder shape for every layer

- My mental model: multi-cluster support and no repetition. Every cluster runs
  the same things; a cluster that needs something different (e.g. another
  chart version) should be able to change just that, easily. Above all it has
  to be neat, tidy and simple to find.
- So every layer (`infrastructure/`, `databases/`, `apps/`) follows one rule:
  1. `<layer>/base/<component>/`: the definition, written once.
  2. `<layer>/<cluster>/<component>/`: what this cluster runs from base, plus
     its differences (version, Secret, size).
  3. `clusters/<cluster>/<layer>.yaml`: Flux wiring only (which folder, what
     to wait for). One Flux Kustomization per layer.
- Monitoring is part of infrastructure: `infrastructure/base/monitoring/`.
  cert-manager or Loki would be `infrastructure/base/cert-manager/`,
  `infrastructure/base/loki/`, plus their `infrastructure/devops-cs/<component>/`
  and one line in `infrastructure/devops-cs/kustomization.yaml`.
- No `controllers/` / `configs/` split. That split exists because objects like
  the `VMPodScrape` need a CRD that the chart installs first. Here the
  `VMPodScrape` ships inside the chart (`extraObjects`), and Helm installs a
  chart's CRDs before everything else, so one folder is enough.
- `apps` waits for `databases` and `infrastructure`, like in Flux's example:
  anything apps may need (monitoring now, cert-manager later) is up before
  they deploy. Trade-off, accepted: if the monitoring install breaks, new app
  deploys wait until it is fixed; running apps keep running. (The final
  review flagged this when monitoring sat inside `infra-controllers`.)
- Checked: the real chart 0.93.0 renders the `VMPodScrape` from our values, and
  a version patch in a cluster overlay changes only that cluster.

## Change 5: Sloth defaults, upstream Sloth dashboards

- I don't want hand-rolled or edited dashboards. Sloth's own dashboards were
  committed with edits because our SLO period was 4 weeks and they assume
  Sloth's default 30 days.
- So: keep Sloth's defaults. The SLO period is 30 days (retention is the
  chart's default, one month = 31 days),
  and the two Sloth dashboards run exactly as published. The only change is
  the data source, because we use VictoriaMetrics.
- Grafana downloads them from grafana.com at every start (pinned revisions in
  `helmrelease.yaml`, `grafana.dashboards`); the chart fills in the data
  source. Trade-off, accepted: if grafana.com is unreachable when Grafana
  starts, those two dashboards are missing until the next start.
- No tool generates a dashboard per SLO outside Grafana Cloud; Sloth and Pyrra
  both ship generic dashboards that find every SLO by its labels.

## Change 6: Sloth controller instead of the Sloth CLI

- Running the Sloth CLI by hand and committing its output doesn't scale:
  imagine hundreds of microservices in a monorepo pushing to this GitOps repo.
  Every SLO change would need a generation step, and every Sloth upgrade a
  commit touching every service's rules.
- So: the Sloth controller runs in the cluster. Each app ships its SLOs as a
  `PrometheusServiceLevel` (`apps/base/<app>/slo.yaml`) next to its other
  manifests; nothing is generated or committed by hand. Sloth version, period
  and plugin chain are set once, in `infrastructure/base/monitoring/sloth.yaml`.
- Sloth writes `PrometheusRule`s (prometheus-operator's kind), and the
  VictoriaMetrics operator converts them into `VMRule`s. That needs the
  `PrometheusRule` CRD (only that one) and the operator's owner references, so
  deleting an SLO also deletes its rules.
- Sloth's status has no conditions, so Flux would call a failed SLO Ready. The
  `apps` Kustomization checks Sloth's `promOpRulesGenerated` flag instead.
- Trade-off, accepted: the generated rules are no longer in git (like the Helm
  charts, which Flux also renders in the cluster).

## Change 7: open questions settled

- Only published dashboards. The two hand-written Overview boards (`apps`,
  `platform`) are deleted: I don't want hand-rolled dashboards. App health is
  the Sloth SLO dashboards, depth is the Components drill-downs, platform
  problems become alerts, and anything else is a query in Grafana Explore.
- Postgres stays on `emptyDir`: its data is throwaway here, and the backend's
  500s after a restart are an application defect (known issue below).
- No CI for now: I push straight to `main`, and Flux reports a broken render a
  minute later. Add CI once changes go through pull requests.
- `bootstrap/bootstrap.sh` waits for the `apps` Kustomization instead of the
  three Deployments. Those don't exist yet when `flux bootstrap` returns, so
  `kubectl wait` failed at once and ended the script. Tested with a full
  rebuild on 2026-09-27 (cluster deleted, script run once): exit 0 after about
  4.5 minutes, everything Ready, no manual step.

## Change 8: alerting

- SLO targets: 99% for all four SLOs. ml-api's availability is measured by
  only 120 probes an hour; at 99.9% two failed probes in an hour would page.
  At 99% a page needs about 9 minutes of full outage in an hour. Latency
  thresholds stay at 1 s and 0.25 s.
- Sloth now also generates the burn-rate alerts (page and ticket per SLO).
- Platform alerts come from the chart's published default rules, not
  hand-written ones: its sync job downloads them at every Helm upgrade, pinned
  to the versions that run here, plus the postgres-exporter mixin. Flux
  failures come from Flux's own notification-controller, sent to Alertmanager.
  Two rules are off with a reason (in `helmrelease.yaml` and the devops-cs
  overlay). Details: `docs/superpowers/specs/2026-09-27-alerting-design.md`.
- No notification channel yet: alerts are only visible in the Alertmanager UI
  and Grafana.
- **Secrets are plain text in git** (`databases/devops-cs/postgres/secret.yaml`,
  `apps/devops-cs/backend-api/secret.yaml`). I left it that way for now. A
  channel's webhook URL or password must not be added like that: encrypt
  Secrets with SOPS + age (Flux decrypts them natively) first.

## Change 9: logs

- VictoriaLogs from the same VictoriaMetrics chart: VLAgent on every node reads
  every container's log, VLSingle stores it for a month (chart defaults). One
  vendor, one operator, no second stack.
- Logs are viewed in Grafana (Explore, data source `VictoriaLogs`) and in
  VictoriaLogs' own UI. The Grafana plugin is downloaded from grafana.com at
  every Grafana start, in the background: without internet Grafana still
  starts, only the logs data source is missing.
- Most logs are plain text, so Grafana showed them as level unknown.
  VictoriaLogs has no built-in detection for that (open upstream issue), so
  the data source has one regex per level covering the formats we run. Chosen
  over Loki: Loki's own detection misses `ERROR:` lines, and it needs a second
  chart and its own agent (Grafana Alloy).
- **Preferred fix: the apps log JSON** (one JSON object per line with `level`
  and `message`). It is the standard way: vlagent turns JSON fields into log
  fields by itself, so levels work everywhere, a traceback stays one entry,
  and there are no level rules to maintain and no log shipper to configure.
  Not done here because the app images aren't ours; until then the level
  rules stay, and a split traceback is read back with
  `… "Traceback" | stream_context after 30`.
- Alerts only on the logging stack's own health (published rules, pinned); no
  alerts on log content.

## Change 10: SLOs from the apps' own metrics, no blackbox

- Both apps have both SLOs (availability and latency), generated by Sloth
  from the metrics each app exposes itself. That was the intent from the
  start; ml-api's availability had used a blackbox `POST /predict` probe
  instead.
- The blackbox exporter is gone. Its other checks (`/health`, `/ready`) fed
  no alert or dashboard; the kubelet already calls those endpoints and the
  default alerts (`KubePodNotReady`, `KubePodCrashLooping`, …) fire on them.
- Checked in the running code and `/metrics`: ml-api's `/predict` handler has
  no error path and only counts status 200, so its availability SLO reads 0
  errors until the app records the real response status. Requests that never
  reach a pod aren't counted by either app; the pod alerts cover that.

## Where things live

| I want to… | Go to |
|---|---|
| See how a component is defined | `<layer>/base/<component>/` |
| Change something for one cluster (version, Secret, size) | `<layer>/<cluster>/<component>/` |
| See what a cluster runs and in what order | `clusters/<cluster>/` (`flux-system/` is Flux itself) |
| Add an infrastructure component (cert-manager, Loki) | `infrastructure/base/<component>/` + `infrastructure/<cluster>/<component>/` + one line in `infrastructure/<cluster>/kustomization.yaml` |
| Add a cluster | `clusters/<cluster>/` + a `<layer>/<cluster>/` overlay per layer |
| Add a new top-level layer folder | Also add `!/<folder>` to `.sourceignore`; Flux only downloads the folders listed there (the `databases/` layer was missing at first: "kustomization path not found") |
| Add or change a dashboard | `infrastructure/base/monitoring/dashboards/`: the JSON file plus one `configMapGenerator` entry in its `kustomization.yaml`. Sloth's SLO dashboards: grafana.com ID and revision in `helmrelease.yaml` (`grafana.dashboards`) |
| Use a different dashboard on one cluster | `infrastructure/<cluster>/monitoring/kustomization.yaml`: a `configMapGenerator` entry with the dashboard's name, `namespace: monitoring`, `behavior: replace` and a file with the same name |
| Add or change an SLO | `apps/base/<app>/slo.yaml` (a `PrometheusServiceLevel`, listed in the app's `kustomization.yaml`). Shared Sloth settings: `infrastructure/base/monitoring/sloth.yaml` |
| Change an alert rule | SLO alerts: `apps/base/<app>/slo.yaml` (target) and `infrastructure/base/monitoring/sloth.yaml` (plugin chain). Platform alerts: `defaultRules` in `helmrelease.yaml` (pinned sources; `rules.<AlertName>.enabled: false` to switch one off, per cluster in the overlay) |
| Change where alerts go | `alertmanager.config` in `helmrelease.yaml` (routes, receivers, inhibition) |
| Search logs | Grafana → Explore → `VictoriaLogs`, or `kubectl -n monitoring port-forward svc/vlsingle-victoria-metrics-k8s-stack 9428` and `http://localhost:9428/select/vmui` |

## Known issue: backend 500s after a restart

- Symptom: `POST /process` returns 500 with `relation "documents" does not
  exist`.
- Cause: postgres stores its data in `emptyDir`, so every restart gives it an
  empty database. backend-api creates the `documents` table only once at
  startup, and `_init_db` swallows errors, so if postgres isn't ready yet the
  table is never created.
- `dependsOn` doesn't fix this. It only sets the order of Flux reconciliation.
  It does nothing when Kubernetes restarts pods, e.g. after a cluster restart.
- The proper fix is in the application (retry table creation, or create it on
  connect). That is out of scope here.
- Workaround for now: once postgres is running, restart the backend:
  `kubectl -n backend-api rollout restart deployment/backend-api`
