# Step 0 Environment Inspection Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Run the supplied case-study environment and document its actual metrics before designing monitoring.

**Architecture:** Use the existing bootstrap and workloads without application changes. Inspect each API pod through local port-forwarding and save evidence; do not deploy a collector or monitoring stack.

**Tech Stack:** Docker, k3d v5+, kubectl, Flux CLI v2+, GitHub, curl.

**Spec:** [Approved Step 0 spec](../specs/2026-09-25-step-0-environment-inspection-design.md)

## Global Constraints

- Use an explicit case-study Kubernetes context for subsequent commands.
- Step 0 does not inject failures, restart workloads, scale deployments, or change traffic.
- Stop temporary port-forwards after inspection; retain the intended case-study cluster.
- Exclude tokens, Secret contents, sensitive payloads, and unfiltered logs.
- Step 0 does not choose SLO targets, latency cutoffs, error-rate thresholds, burn rates, or alert evaluation windows.
- Keep the supplied application manifests unchanged. Obtain explicit approval before deleting an existing cluster or changing the baseline.

## Files and deliverables

- Read: `bootstrap/bootstrap.sh`, `bootstrap/k3d.config.yaml`, `clusters/devops-cs/`, and the four application directories under `apps/`.
- Bootstrap-generated: Flux manifests under `clusters/devops-cs/`; inspect the actual generated files after bootstrap rather than authoring replacements.
- Create after inspection: `docs/inspection/step-0-findings.md`, the report.
- Create after inspection: `docs/inspection/step-0-metrics.txt`, sanitized raw samples with UTC timestamps and pod identities.
- No permanent collector, parser, test suite, or application changes. Runtime checks provide the evidence for this investigation.

## Review Focus

1. Existing `devops-cs`: inspect and reuse a matching cluster; never allow bootstrap to delete it without approval. Check in Task 1.
2. Wrong context or fork: verify context, Git source URL, branch, and revision before inspection. Check in Tasks 1–2.
3. Missing credentials or incompatible images: report the exact blocker without exposing secrets or swapping images. Check in Task 1 and confirm runtime behavior in Task 2.
4. Missing series, pod replacement, or counter resets: retain timestamps and pod identity; do not calculate deltas across resets. Check in Task 3.
5. Synthetic or unexercised request paths: report coverage limits rather than claim end-to-end reliability. Check in Tasks 3–4.

## Task 1: Prepare the environment safely

- [ ] Read the approved spec and bootstrap script. Work from the repository root. Inspect local changes before any Git operation and preserve unrelated work.
- [ ] Check installed tools and Docker capacity:

```sh
command -v docker k3d kubectl flux gh brew
docker version
docker info --format 'OS={{.OSType}} arch={{.Architecture}} CPUs={{.NCPU}} memory={{.MemTotal}}'
kubectl version --client
```

- [ ] Install only missing prerequisites using the existing package manager. On this Homebrew machine, use `brew install k3d` and/or `brew install fluxcd` only if absent. Confirm `k3d version` and `flux version --client` meet the brief. If Docker is not running, start the installed Docker application and repeat the daemon check.
- [ ] Check GitHub access without printing tokens:

```sh
gh auth status
git remote get-url origin
git ls-remote --symref origin HEAD
kubectl config get-contexts
k3d cluster list
```

Use the fork's default branch unless the user has selected another branch. Set `BRANCH` to that observed name. Check whether `GITHUB_TOKEN` is present without revealing its value. If needed, use an authenticated GitHub CLI credential only inside the bootstrap process environment; otherwise request interactive authentication. Do not run `gh auth token` as a standalone tool call that prints its output.

- [ ] Inspect registry manifests for the three supplied images:

```sh
docker manifest inspect --verbose ghcr.io/voize-gmbh/devops-case-study/ml-api:v1
docker manifest inspect --verbose ghcr.io/voize-gmbh/devops-case-study/backend-api:v1
docker manifest inspect --verbose ghcr.io/voize-gmbh/devops-case-study/load-generator:v1
```

Compare supported platforms with Docker's platform. If native support is absent, investigate existing emulation support; do not assume compatibility or change the supplied image. Report registry/authentication failures separately from architecture failures.

- [ ] If `devops-cs` exists, inspect its nodes, namespaces, workloads, and Flux source using its explicit context. Reuse only a matching case-study environment. If unsuitable, stop for replacement approval; never run the destructive bootstrap against it.
- [ ] Explain the bootstrap's remote effects before running it: Flux can commit generated manifests to the fork and configure a deploy key. Confirm the intended fork and branch. Keep tokens out of command output and evidence.

**Check:** Record tool versions, Docker platform/capacity, repository/branch, and the cluster decision. An unresolved prerequisite blocks Task 2; it is not a passing check.

## Task 2: Bootstrap or verify the supplied environment

- [ ] For a new cluster only, run the existing script from the repository root with a credential in its environment:

```sh
./bootstrap/bootstrap.sh https://github.com/slobra1n/devops-case-study "$BRANCH"
```

For a matching existing cluster, skip creation and inspect its reconciliation. If repair would change its configuration, explain the proposed repair and obtain approval first.

- [ ] Set `CTX` to the verified case-study context (`k3d-devops-cs` for the normal new-cluster path). Verify the context before continuing:

```sh
kubectl --context="$CTX" get nodes -o wide
flux --context="$CTX" get all -A
kubectl --context="$CTX" -n flux-system get gitrepository flux-system -o yaml
```

Record the source URL, branch, and artifact revision. Do not inspect Secret contents. If the script exits early, inspect actual cluster and Flux state before taking action; do not rerun a script that now would delete the cluster.

- [ ] Wait for all four deployments, then check ready replica counts rather than relying only on the Available condition:

```sh
kubectl --context="$CTX" -n postgres rollout status deployment/postgres --timeout=180s
kubectl --context="$CTX" -n ml-api rollout status deployment/ml-api --timeout=180s
kubectl --context="$CTX" -n backend-api rollout status deployment/backend-api --timeout=180s
kubectl --context="$CTX" -n load-generator rollout status deployment/load-generator --timeout=180s
kubectl --context="$CTX" get deployments,pods -A
```

The APIs must each have two ready replicas; PostgreSQL and the generator must each have one. Investigate readiness failures, restarts, or image-pull problems through pod descriptions, bounded logs, and relevant namespace events (`kubectl --context="$CTX" -n "$NS" get events --sort-by=.metadata.creationTimestamp`). Preserve the baseline and ask before fixes that change it.

**Check:** Save sanitized health/reconciliation evidence and actual image IDs. An image manifest inspection alone does not prove the images run. Proceed only when the workloads run and Flux has no unresolved reconciliation failures.

## Task 3: Inspect metrics and normal traffic

- [ ] Discover pod names and record UIDs, restart counts, start times, and image IDs:

```sh
kubectl --context="$CTX" -n ml-api get pods -l app=ml-api -o json
kubectl --context="$CTX" -n backend-api get pods -l app=backend-api -o json
```

- [ ] Start four managed, localhost-bound port-forwards. Assign ports 18001 and 18002 to the two ML pods, and 18003 and 18004 to the two backend pods. Set `NS`, `POD`, and `PORT` from the discovered pod-to-port mapping before each command:

```sh
kubectl --context="$CTX" -n "$NS" port-forward --address=127.0.0.1 "pod/$POD" "$PORT:8000"
```

Keep the process handles so only these forwards are stopped afterward. If a port is occupied, select another free local port and record the mapping; do not kill unrelated processes.

- [ ] Fetch each pod's health and metrics. Save timestamped samples with pod identity:

```sh
curl --fail --silent --show-error --max-time 10 "http://127.0.0.1:$PORT/health"
date -u +%Y-%m-%dT%H:%M:%SZ
curl --fail --silent --show-error --max-time 10 "http://127.0.0.1:$PORT/metrics"
```

- [ ] Sample all four pods at the start, approximately 60 seconds later, and approximately 120 seconds after the start. Record actual per-fetch times, not assumed exact intervals. This short observation period checks metric behavior, not reliability targets. Recheck pod identities and restart counts afterward. If counters decrease or a pod changes, discard that interval for delta calculations and collect a fresh pair from a stable pod.
- [ ] Inventory the application families from the brief: `ml_api_requests_total`, `ml_api_request_duration_seconds`, `ml_api_predictions_total`, `ml_api_memory_bytes`, `backend_api_requests_total`, `backend_api_request_duration_seconds`, `backend_api_db_connections_active`, and `backend_api_db_queries_total`. Include additional application families and relevant runtime families found live. Record HELP/TYPE, units, labels, observed statuses, and histogram boundaries. Mark absent families or unexercised series explicitly.
- [ ] Inspect a bounded slice of generator logs:

```sh
kubectl --context="$CTX" -n load-generator logs deployment/load-generator --since=5m --tail=100
```

Use discovered API documentation or bounded application logs if available to explain routes and outcomes; do not invent business-request payloads. Compare logs with endpoint/status counter deltas. Check for probe and scrape endpoints in request metrics. Record uncertainty if the evidence cannot establish causality, timing semantics, or an end-to-end flow.
- [ ] Preserve sanitized raw samples in `docs/inspection/step-0-metrics.txt` with the pod-to-port mapping and timestamps. Include only metric exposition and observation metadata, not unfiltered logs. Stop the four managed forwards; leave the cluster running.

**Check:** Each API replica has timestamped samples, metric metadata, and either valid observed deltas or an explicit explanation of why an interval is unusable. Verify histogram bucket ordering and count/sum availability from exposition; do not estimate a latency objective from this sample.

## Task 4: Write and review the findings

- [ ] Write the report with five sections: environment and revisions; Flux/workload health; metric inventory; observed traffic and deltas; candidate measurable outcomes and gaps. Link the sanitized raw samples and meet the evidence requirements in the approved spec.
- [ ] Check the report against all seven spec acceptance criteria. Mark missing evidence and unresolved runtime failures as blockers; findings review remains pending until the user completes it.
- [ ] Review changes for credentials, sensitive samples, accidental application changes, and unrelated user work. Commit only approved Step 0 documentation and sanitized evidence. Inspect bootstrap-generated changes without rewriting or force-pushing remote history; do not push additional local commits without authorization.
- [ ] Present the findings for user review before beginning monitoring design.

Execute sequentially after plan approval.
