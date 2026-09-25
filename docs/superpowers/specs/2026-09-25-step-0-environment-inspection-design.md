# Step 0: Set up and inspect the supplied environment

## Purpose and approval boundary

Prepare the Voize case-study environment and gather evidence for the monitoring design. The candidate must be able to explain the running system, the metrics it exposes, and the limits of those measurements during the interview.

The user approved setup and inspection before monitoring deployment. Obtain approval of this written spec, then the implementation plan, before execution. Obtain findings review before designing monitoring.

VictoriaMetrics remains the selected metrics platform. The user selected a trimmed VictoriaMetrics Kubernetes stack for the later monitoring stage. Centralized logging follows completion of the required monitoring. Neither deployment belongs to Step 0.

Step 0 does not choose SLO targets, latency cutoffs, error-rate thresholds, burn rates, or alert evaluation windows.

## Repository facts

From repository inspection, not a running cluster:

- Fork: `git@github.com:slobra1n/devops-case-study.git`; baseline: `b15f364` (`Initial setup`).
- Entry point: `bootstrap/bootstrap.sh`. It accepts an HTTPS repository URL, requires `k3d`, `kubectl`, `flux`, and `GITHUB_TOKEN`, and invokes Flux bootstrap against the fork.
- Bootstrap deletes an existing `devops-cs` cluster. Check before running it.
- PostgreSQL uses `emptyDir`; pod replacement can discard its data.
- The initial executable check found Docker and kubectl, but not k3d or Flux on PATH. Runtime health and credentials remain unchecked.

## Scope

### 1. Check prerequisites and protect existing resources

Inspect the available tool versions, Docker daemon health and capacity, local cluster inventory, Kubernetes contexts, and access to the fork and container images. Check architecture compatibility rather than assuming the supplied images run on this machine.

Install missing prerequisites through the machine's existing package-management approach after plan approval. Check for a usable GitHub credential without printing its value or storing it in Git, shell examples, or the evidence report. Request user action if authentication requires it.

Before running bootstrap, check for `devops-cs`. Do not run the script against an existing cluster because it deletes that cluster. Inspect an existing cluster for a matching case-study setup and reuse it if appropriate. If reuse is not possible, obtain explicit approval before replacement. Do not delete or modify unrelated clusters.

Use an explicit case-study Kubernetes context for subsequent commands. Document the intended GitHub changes from Flux bootstrap before execution. Do not change application behavior, resource settings, credentials, or traffic configuration merely to make setup pass.

### 2. Bootstrap and verify the supplied workloads

For a new environment, use the provided bootstrap path with `https://github.com/slobra1n/devops-case-study` and the agreed branch. Confirm the branch before bootstrap rather than assuming it.

Verify all supplied deployments, including the load generator, and inspect readiness, restarts, image-pull failures, and relevant events. Check both replicas of each API. Confirm that Flux points to the intended fork and branch and has reconciled the intended revision.

Distinguish successful installation from a healthy environment. A bootstrap process exit code alone does not prove that workloads or Flux are healthy. Investigate failures without changing the supplied application baseline. If a fix requires such a change, document the evidence and seek approval before making it.

### 3. Inspect live metrics and traffic

Inspect `/metrics` from each API pod through localhost-bound port-forwarding so that requests to a Service do not hide differences between replicas. Capture pod identity, timestamps, and restart state with the observations.

Preserve sanitized raw exposition from each replica. In the report, explain the application metric families named in the brief, additional application metrics, and runtime metrics relevant to diagnosis:

- Record names, types, HELP text, units, label keys and observed values.
- Record histogram boundaries, counts, and sums.
- Record changes under traffic and series that appear only after relevant requests.
- Mark listed-but-unobserved families as unobserved, not zero.

Collect repeated timestamped samples during a recorded normal-traffic observation period. Use counter and histogram deltas to describe that period, handling counter resets and pod replacements explicitly. Record the sampling interval and duration. A short synthetic baseline cannot establish production reliability or justify a production SLO.

Inspect load-generator logs and application responses where the interfaces are discoverable. Determine which endpoints and outcomes the generator exercises and whether it calls the two APIs independently or represents an end-to-end workflow. Do not invent request payloads or assume that an HTTP success proves correct transcription or document processing.

Check whether probes and metric scrapes contribute to request metrics. Distinguish successful business requests, validation failures, server failures, and transport failures where the available evidence allows it. Do not infer undocumented status semantics from label names alone.

Step 0 does not inject failures, restart workloads, scale deployments, or change traffic. Report unexercised error paths and missing measurements as limits for the next design stage. Stop temporary port-forwards after inspection; retain the intended case-study cluster.

### 4. Record the evidence and its limits

After inspection, write `docs/inspection/step-0-findings.md` with environment details, Flux and workload health, the metric inventory, traffic observations, candidate measurable outcomes, and measurement gaps.

Include the Git revision, image references and available image IDs, pod identities, timestamps, sampling details, reproducible commands, and compact supporting excerpts. Separate observations, documented behavior, and inferences.

Exclude tokens, Secret contents, sensitive payloads, and unfiltered logs. Do not imply inspection has happened before it has.

## SLO design after Step 0

Define user-facing SLIs and their measurement limits before agreeing on objectives and error-budget burn-rate alerts. Use the inspection evidence and user needs, not copied numerical examples or baseline performance alone.

- [Google SRE: Service Level Objectives](https://sre.google/sre-book/service-level-objectives/)
- [Google SRE Workbook: Implementing SLOs](https://sre.google/workbook/implementing-slos/)
- [Google SRE Workbook: Alerting on SLOs](https://sre.google/workbook/alerting-on-slos/)

## Out of scope

- Deploying VictoriaMetrics, Grafana, exporters, or alerting components.
- Choosing dashboard panels, writing alert rules, or setting numerical reliability targets.
- Deploying centralized logging or tracing.
- Modifying application images, instrumentation, traffic settings, or database storage.
- Claiming end-to-end reliability or inference correctness from API status counters alone.
- Emailing the interview deliverable or publishing unreviewed implementation changes.

## Acceptance criteria

Step 0 is complete when:

1. The intended case-study cluster runs the supplied workloads, including both replicas of each API and the load generator. Health and any observed restarts have supporting evidence.
2. Flux reconciles the intended fork, branch, and revision without unresolved reconciliation failures.
3. The report documents per-replica metric exposition and repeated samples under normal supplied traffic, with timestamps and reset handling.
4. The inventory covers metric types, labels, histogram boundaries, and which listed metrics or series remain unobserved.
5. The report explains the observed traffic flow and explicitly records gaps that affect candidate SLIs.
6. No monitoring stack, numerical SLO targets, or alert rules have been introduced.
7. The user has reviewed the findings before the next monitoring design begins.

An unavailable prerequisite, unsupported image, or inaccessible runtime is a blocker, not a passing result. Record the exact failure and attempts to resolve it; finish any independent inspection that remains possible without claiming Step 0 complete.

