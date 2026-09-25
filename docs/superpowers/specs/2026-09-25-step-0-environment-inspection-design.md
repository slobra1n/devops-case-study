# Step 0: Set up and inspect the supplied environment

## Purpose and approval boundary

Prepare the Voize case-study environment and gather evidence for the monitoring design. The candidate must be able to explain the running system, the metrics it exposes, and the limits of those measurements during the interview.

The user approved this scope: bootstrap the supplied environment and inspect it before deploying monitoring. This document requires user review before an implementation plan is written. Execution requires separate approval of that plan.

VictoriaMetrics remains the selected metrics platform. The user selected a trimmed VictoriaMetrics Kubernetes stack for the later monitoring stage. Centralized logging follows completion of the required monitoring. Neither deployment belongs to Step 0.

Earlier proposed numerical alert thresholds are withdrawn. Step 0 does not choose SLO targets, latency cutoffs, error-rate thresholds, burn rates, or alert evaluation windows.

## Repository facts

These facts come from repository inspection, not a running cluster:

- Fork: `git@github.com:slobra1n/devops-case-study.git`.
- Starting commit inspected: `b15f364` (`Initial setup`).
- `bootstrap/k3d.config.yaml` defines one k3s server and no agent nodes, with Traefik disabled.
- `bootstrap/bootstrap.sh` requires `k3d`, `kubectl`, `flux`, and `GITHUB_TOKEN`. It accepts an HTTPS GitHub repository URL, not the SSH clone URL.
- The bootstrap script deletes an existing cluster named `devops-cs` before creating one. It also invokes `flux bootstrap github`, which can write Flux manifests and configure access to the fork.
- Both APIs have two replicas, a named `http` Service port on port 8000, and health and readiness probes.
- The load generator has separate ML and backend URLs and `REQUEST_INTERVAL_MS=2000`. The actual request mix and sequence require runtime inspection.
- PostgreSQL uses an `emptyDir` volume. Pod replacement can discard its data.
- Flux reconciles `infrastructure/controllers`, then `infrastructure/configs`. Applications depend on `infra-controllers`. Both infrastructure folders currently have empty resource lists.
- The inspected checkout contains deployment manifests but no application source. The brief lists API metrics; live metric exposition must confirm their shape and behavior.
- The initial local executable check found Docker and kubectl, but not k3d or Flux on PATH. This does not establish Docker daemon health, cluster state, or credential availability.

Relevant files:

- `bootstrap/bootstrap.sh`
- `bootstrap/k3d.config.yaml`
- `clusters/devops-cs/infrastructure.yaml`
- `clusters/devops-cs/apps.yaml`
- `apps/ml-api/deployment.yaml`
- `apps/ml-api/service.yaml`
- `apps/backend-api/deployment.yaml`
- `apps/backend-api/service.yaml`
- `apps/load-generator/deployment.yaml`
- `apps/postgres/deployment.yaml`

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

For each exposed metric family, record:

- Name, declared type, HELP text, and unit where available.
- Label keys and observed label values, including endpoints and statuses.
- Histogram bucket boundaries, count, and sum where present.
- Whether the family or individual series appears only after relevant traffic.
- Evidence of changes under the supplied traffic, or the absence of such evidence.

Cover the metric families named in the case-study brief and record additional families the applications expose. Mark listed-but-unobserved families as unobserved, not as zero.

Collect repeated timestamped samples during a recorded normal-traffic observation period. Use counter and histogram deltas to describe that period, handling counter resets and pod replacements explicitly. Record the sampling interval and duration. A short synthetic baseline cannot establish production reliability or justify a production SLO.

Inspect load-generator logs and application responses where the interfaces are discoverable. Determine which endpoints and outcomes the generator exercises and whether it calls the two APIs independently or represents an end-to-end workflow. Do not invent request payloads or assume that an HTTP success proves correct transcription or document processing.

Check whether probes and metric scrapes contribute to request metrics. Distinguish successful business requests, validation failures, server failures, and transport failures where the available evidence allows it. Do not infer undocumented status semantics from label names alone.

Step 0 does not inject failures, restart workloads, scale deployments, or change traffic. Report unexercised error paths and missing measurements as limits for the next design stage. Stop temporary port-forwards after inspection; retain the intended case-study cluster.

### 4. Record the evidence and its limits

Write `docs/inspection/step-0-findings.md` after running the inspection. Include:

- Environment and tool versions, image references and available image IDs, Git revision, cluster/context, and observation timestamps.
- Flux reconciliation and workload-health evidence.
- The live metric inventory, label semantics, and histogram boundaries.
- Reproducible commands and compact, sanitized output excerpts that support each finding.
- Observed traffic paths, per-pod counter changes, and sampling details.
- Candidate user-facing outcomes and the metrics that could measure them.
- Measurement gaps, unresolved semantics, and what would be needed to resolve them.

Separate observed facts, documented behavior, and inferences. Do not include tokens, Secret contents, sensitive payloads, or unfiltered logs. Do not create a findings document that implies inspection has happened before it has.

## SLO design after Step 0

Use Google's SRE guidance for the subsequent design:

1. Identify users and critical journeys. Check whether the supplied demo measures those journeys or only parts of them.
2. Define each service-level indicator as good events divided by eligible events where the metrics support that definition. Document the measurement point, exclusions, and blind spots.
3. Verify that the available histogram boundaries can measure any proposed latency criterion. Percentile graphs may help diagnosis but do not replace a defined good-event ratio for error-budget accounting.
4. Agree on SLO targets and an evaluation period using user needs and the measurement evidence. Treat case-study objectives as explicit assumptions for the exercise, not Voize production commitments.
5. Design error-budget burn-rate alerting, evaluating Google's multiwindow, multi-burn-rate approach against the observed traffic and the chosen objectives. Do not copy the book's numerical examples as service requirements.
6. Address low traffic, no traffic, missing telemetry, failures before requests reach the application, and the difference between synthetic and real user traffic. Keep diagnostic infrastructure signals distinct from user-facing SLOs.

References:

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

## Handoff

After written-spec approval, create a separate implementation plan for Step 0 and obtain approval of that plan before setup begins. After execution and findings review, resume design of the required monitoring with VictoriaMetrics and Google SRE-style SLO alerting. Add centralized logging only after that monitoring meets the case-study requirements.
