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

## Where things live

| I want to… | Go to |
|---|---|
| See how a component is defined | `<layer>/base/<component>/` |
| Change something for one cluster (version, Secret, size) | `<layer>/<cluster>/<component>/` |
| See what a cluster runs and in what order | `clusters/<cluster>/` (`flux-system/` is Flux itself) |
| Add an infrastructure component (cert-manager, Loki) | `infrastructure/base/<component>/` + `infrastructure/<cluster>/<component>/` + one line in `infrastructure/<cluster>/kustomization.yaml` |
| Add a cluster | `clusters/<cluster>/` + a `<layer>/<cluster>/` overlay per layer |
| Add a new top-level layer folder | Also add `!/<folder>` to `.sourceignore`; Flux only downloads the folders listed there (the `databases/` layer was missing at first: "kustomization path not found") |
| Add or change a dashboard | `infrastructure/base/monitoring/dashboards/`: the JSON file plus one `configMapGenerator` entry in its `kustomization.yaml` |
| Use a different dashboard on one cluster | `infrastructure/<cluster>/monitoring/kustomization.yaml`: a `configMapGenerator` entry with the dashboard's name, `namespace: monitoring`, `behavior: replace` and a file with the same name |

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
