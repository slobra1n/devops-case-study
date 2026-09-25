# Step 0 findings — inspection recorded; backend defect unresolved

## Environment and revisions

Observed on 2026-09-25; timestamps below are UTC. [Raw per-replica evidence](step-0-metrics.txt) contains all 12 expositions, individual fetch timestamps, pod UIDs, start times, image IDs, health responses, and before/after states.

- Fork: `git@github.com:slobra1n/devops-case-study.git`; branch: `main`; original remote baseline: `b15f364b08faeebb6f3a7791acf78ed746869be8`.
- GitHub CLI authenticated as `slobra1n`; credential values were not printed or saved in evidence.
- Installed k3d v5.9.0 and Flux CD v2.9.5. Correct Homebrew package: `fluxcd`; the mistakenly installed, unrelated `flux` package was removed.
- kubectl v1.36.1; Kustomize v5.8.1; k3s v1.35.5+k3s1. Docker Desktop 4.88.1, Engine 29.7.2; daemon Linux ARM64, 14 CPUs, 8,318,976,000 bytes RAM (about 7.75 GiB).
- Docker initially failed to answer `info` and Desktop status queries. Logs showed backend communication timeouts. The approved `docker desktop restart --timeout 120` reported failure to stop remaining processes. A subsequent approved force-quit attempt found no matching Docker application processes, so killed none; reopening Docker restored daemon access. No data reset was performed; the original cause is unknown.
- Before bootstrap, `k3d cluster list` was empty. The supplied bootstrap created `devops-cs`; subsequent Kubernetes commands used explicit context `k3d-devops-cs`. No pre-existing cluster was deleted.
- Flux bootstrap created a deploy key and two remote commits: `8fde490c322c38f751ca8579a717aba147613bf1` (components), then `fe37386b32f70e2a86c8188ad1f32c6e5e59f2b4` (sync). Remote changes are confined to three generated files under `clusters/devops-cs/flux-system/`. Remote history was fetched for inspection, not rewritten. Additional local documentation commits are not pushed.

Registry inspection found Linux ARM64 support for all three application images. ARM64 manifest digests were ML `e239bef35e8f6bd85981fdaca8429cb970f9a2349b2947b0c9ddc32440583f7e`, backend `ca275ac8198d72f0650bc2ca31b45f2e2b79c0838607812a0db8c1c4300c485b`, and generator `fca849754b59cdb90dd11c5f50c3336ae0c1bcbe58eb9ee9edb4ddd96aac844d`. These registry digests are distinct from the runtime-reported image IDs below; runtime readiness, not manifest inspection, proves execution.

## Flux and workload health

The bootstrap exited 1 when its immediate rollout check found no `postgres` namespace. Inspection showed Flux's root and infrastructure controller reconciliations ready, with downstream dependencies still settling. Waiting for normal reconciliation resolved this without rerunning bootstrap or changing manifests.

The GitRepository source is `ssh://git@github.com/slobra1n/devops-case-study`, branch `main`, artifact revision `main@sha1:fe37386b32f70e2a86c8188ad1f32c6e5e59f2b4`. The source and all four Kustomizations (`flux-system`, `infra-controllers`, `infra-configs`, `apps`) reported Ready=True at that revision. The node was Ready. All four deployment rollout commands passed.

| Deployment | Ready replicas | Pod suffixes | Restarts before/after |
|---|---:|---|---|
| ML API | 2/2 | `5c9dbbb657-gqs85`, `5c9dbbb657-zqdlx` | 0 / 0 each |
| Backend API | 2/2 | `554cffb7d9-mg4vj`, `554cffb7d9-xgfmw` | 0 / 0 each |
| PostgreSQL | 1/1 | `559474cdd9-mls5n` | 0 / 0 |
| Load generator | 1/1 | `6c49c5965f-2lz2k` | 0 / 0 |

All six workload pods started at `09:20:17Z`; UIDs and container states were unchanged across the observation window. All four directly requested API `/health` responses were HTTP 200 with `{"status":"healthy"}`.

Runtime-reported image IDs (full repository names are in raw evidence):

| Image reference | Runtime digest (`sha256:`) |
|---|---|
| `ghcr.io/voize-gmbh/devops-case-study/ml-api:v1` | `d63e8585292ff3711d89525010a2b3868ba2c8c141e53ffe403d817f2798801a` |
| `ghcr.io/voize-gmbh/devops-case-study/backend-api:v1` | `348f4aa204a84a086bdb430741e7eb9a7029fc7ba648543d34888df6a9580327` |
| `ghcr.io/voize-gmbh/devops-case-study/load-generator:v1` | `892520863b7eeb94949ed5f295122ab0e976ae7e6131a1c780c349b736831111` |
| `postgres:16-alpine` | `721873c34ceb9f8d8fc265984940dc982404c105f19ad51be9fdc5970a6080ea` |

**Application defect remains:** backend `/process` returns 500 despite ready pods and healthy probes. Bounded PostgreSQL logs contained 49 occurrences of `ERROR: relation "documents" does not exist at character 13`. A read-only check returned `voize|public|` for `SELECT current_database(), current_schema(), to_regclass('public.documents');`, confirming the relation is absent. Backend access logs showed `/process` 500 responses, and its DB error counter advanced with those responses. This identifies a missing database relation on the failing path; it does not establish the intended schema or why initialization is absent. No schema, storage, image, or traffic changes were made. Repair needs a reviewed proposal and approval. PostgreSQL uses `emptyDir`; replacing its pod risks losing data.

Reproduction and inspection commands (no Secret reads):

```sh
CTX=k3d-devops-cs
flux --context="$CTX" get all -A
kubectl --context="$CTX" -n flux-system get gitrepository flux-system -o yaml
kubectl --context="$CTX" get deployments,pods -A
kubectl --context="$CTX" -n postgres logs deployment/postgres --since=10m --tail=100
kubectl --context="$CTX" -n postgres exec deployment/postgres -- psql -U voize -d voize -Atc "SELECT current_database(), current_schema(), to_regclass('public.documents');"
```

## Metric inventory

All eight application families named in the brief were present. Values below describe observed series, not every series the images might emit. No business validation failures, ML errors, backend successes, or successful DB query series were observed.

| Family | Type; unit | Exact HELP | Labels and observed values |
|---|---|---|---|
| `ml_api_requests_total` | counter; requests | Total requests | `endpoint=/predict,/health,/ready`; `method=POST,GET`; `status=200` |
| `ml_api_request_duration_seconds` | histogram; seconds | Request latency | `endpoint=/predict`; bucket `le` below; no status label |
| `ml_api_predictions_total` | counter; predictions | Total predictions made | none |
| `ml_api_memory_bytes` | gauge; bytes | Simulated memory usage in bytes | none; 0 in all samples; not actual RSS |
| `backend_api_requests_total` | counter; requests | Total requests | `/process,POST,500`; `/health,GET,200`; `/ready,GET,200` for `endpoint,method,status` |
| `backend_api_request_duration_seconds` | histogram; seconds | Request latency | `endpoint=/process`; bucket `le` below; no status label |
| `backend_api_db_connections_active` | gauge; connections | Active database connections | none; 0 at each sample, not proof of no intervening connections |
| `backend_api_db_queries_total` | counter; queries | Total DB queries | `status=error` |

Both histograms expose cumulative buckets at `0.01, 0.05, 0.1, 0.25, 0.5, 1.0, 2.5, 5.0, 10.0, +Inf` seconds, plus `_count` and `_sum`. Checks across all 12 expositions confirmed ordered boundaries, nondecreasing bucket counts, and `+Inf == _count`. Histograms only showed the business endpoints, while request counters also included probes. Backend failure requests increment the histogram; a low measured backend latency here describes fast failures, not successful processing. Timing boundaries are not established by the HELP text or OpenAPI documentation.

Additional application families: `ml_api_requests_created`, `ml_api_request_duration_seconds_created`, `ml_api_predictions_created`, `backend_api_requests_created`, `backend_api_request_duration_seconds_created`, and `backend_api_db_queries_created`. All are gauges with the same HELP and non-`le` labels as their corresponding parent families; observed values are epoch timestamps, not rates or request counts.

Runtime families appeared on both APIs:

| Family | Type; unit | Exact HELP | Labels |
|---|---|---|---|
| `python_gc_objects_collected_total` | counter; objects | Objects collected during gc | `generation=0,1,2` |
| `python_gc_objects_uncollectable_total` | counter; objects | Uncollectable objects found during GC | `generation=0,1,2` |
| `python_gc_collections_total` | counter; collections | Number of times this generation was collected | `generation=0,1,2` |
| `python_info` | gauge; information marker | Python platform information | `implementation=CPython`, `major=3`, `minor=11`, `patchlevel=15`, `version=3.11.15` |
| `process_virtual_memory_bytes` | gauge; bytes | Virtual memory size in bytes. | none |
| `process_resident_memory_bytes` | gauge; bytes | Resident memory size in bytes. | none |
| `process_start_time_seconds` | gauge; epoch seconds | Start time of the process since unix epoch in seconds. | none |
| `process_cpu_seconds_total` | counter; CPU seconds | Total user and system CPU time spent in seconds. | none |
| `process_open_fds` | gauge; descriptors | Number of open file descriptors. | none |
| `process_max_fds` | gauge; descriptors | Maximum number of open file descriptors. | none |

These process-local metrics can aid diagnosis but do not replace container/node measurements. Kubernetes namespace/pod labels are not present in the application exposition; the raw evidence records identity separately. No infrastructure monitoring was added.

## Observed traffic and deltas

The supplied generator is configured with both service URLs and `REQUEST_INTERVAL_MS=2000` ([deployment](../../apps/load-generator/deployment.yaml)). Its bounded logs showed alternating POSTs to ML `/predict` and backend `/process`, backend HTTP 500 errors, and earlier startup connection-refused errors. Logs were fetched with `--since=5m --tail=100`; this is bounded evidence, not a complete request history. API access logs independently identify backend failures. No transport failures can be counted from server-side request counters alone.

Live `/openapi.json` documents ML `/predict` and backend `/process`, plus `/health`, `/ready`, and `/metrics`. Business request schemas are unspecified and response schemas empty; the descriptions alone do not establish payload validation or semantic correctness. The generator directly targets both APIs. No evidence establishes that an ML response feeds a backend request, or that one backend operation invokes ML; an end-to-end transcription/document journey remains unproven.

Four localhost-only forwards mapped ML `gqs85`→18001, ML `zqdlx`→18002, backend `mg4vj`→18003, backend `xgfmw`→18004. For each pod, the pattern was:

```sh
kubectl --context=k3d-devops-cs -n "$NS" port-forward --address=127.0.0.1 "pod/$POD" "$PORT:8000"
curl --fail --silent --show-error --max-time 10 "http://127.0.0.1:$PORT/health"
date -u +%Y-%m-%dT%H:%M:%SZ
curl --fail --silent --show-error --max-time 10 "http://127.0.0.1:$PORT/metrics"
```

Collection used equivalent bounded HTTP GETs with UTC timestamps. A first collection tool timed out and produced no retained evidence; it is excluded from calculations. The retained run sampled each replica three times, near `09:24:05`, `09:25:05`, and `09:26:05`. Individual first-to-last intervals were 120.004776, 120.010779, 120.011096, and 120.011095 seconds in the mapping order above. Scrapes are sequential, not a single atomic cluster snapshot. All counters were nondecreasing in both intervals, and all pod identities/restarts remained stable; no retained interval needed discarding.

| Pod suffix | Business counter at t0 / t1 / t2 | HTTP status | Delta | Histogram sum delta (seconds) | Histogram count delta |
|---|---|---|---:|---:|---:|
| ML `gqs85` | 36 / 44 / 54 | 200 | 18 | 4.215845084 | 18 |
| ML `zqdlx` | 23 / 32 / 39 | 200 | 16 | 3.494174462 | 16 |
| Backend `mg4vj` | 28 / 35 / 44 | 500 | 16 | 0.010106166 | 16 |
| Backend `xgfmw` | 27 / 37 / 45 | 500 | 18 | 0.019453956 | 18 |

- ML: 34 HTTP-success requests and 34 predictions; 22 fell in the cumulative ≤0.25-second bucket and all 34 in ≤0.5 seconds. These are observations, not latency objectives or proof of correct inference.
- Backend: 34 failed requests and 34 DB errors; all 34 fell in ≤0.01 seconds. No successful backend business series was present.
- Every API replica added 24 `/ready` and 12 `/health` requests, all status 200. Probe traffic therefore materially inflates an unfiltered success ratio. The inspector also requested health before the retained window.
- `/metrics` and `/openapi.json` had no request-counter series in any retained sample despite successful inspector GETs; probe endpoints did. This establishes observed exclusion, not a universal claim about all instrumentation paths.
- The zero-valued simulated ML memory and point-in-time DB connection gauges must not be treated as measured real memory or proof of no DB activity.

All four managed forwards were stopped after inspection. The case-study cluster remains running; workloads were not restarted, scaled, or subjected to injected failures.

## Candidate measurable outcomes and gaps

Candidates for later discussion, not an SLO design:

- Per-endpoint HTTP outcome ratios are measurable with request counters, excluding probe/scrape endpoints. Server counters omit pre-server failures and do not establish business correctness.
- Per-endpoint latency distributions are measurable at the existing bucket boundaries. Histograms lack status labels, so a successful-request-only latency distribution cannot be recovered from these samples. Instrumentation timing boundaries still need confirmation.
- Predictions and DB query outcomes provide supporting signals, not proven end-to-end journey completion. No evidence measures transcription quality, durable document completion, or user-perceived latency.
- Repair and reinspection of the backend success path are prerequisites for a healthy normal-traffic baseline. The intended database schema and initialization contract must be established before changing anything; do not invent a table definition from its name.
- This short synthetic observation on one local node cannot establish production reliability, capacity, production objectives, or alert thresholds.

Acceptance review against the approved spec:

1. All supplied deployments and both API replicas run, with health and restart evidence. **Business-path health remains blocked by the missing relation**; readiness alone is not a healthy baseline.
2. Flux reconciles the intended fork, branch, and revision with no unresolved reconciliation failures: observed.
3. Three timestamped samples per API replica, stable identities, and reset checks: recorded.
4. Types, HELP, units, labels, buckets, and unobserved outcome series: recorded.
5. Observed traffic and candidate-SLI limitations: recorded; end-to-end causality remains unproven.
6. No monitoring stack, numerical SLO targets, or alert rules introduced: preserved.
7. **User findings review is pending.** Monitoring design has not begun.

Step 0 is not declared complete. Review these findings and approve a separately scoped backend repair proposal before baseline changes; monitoring design remains behind the findings-review gate.
