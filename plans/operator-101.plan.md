---
name: Operator-101 zombie-job operator
overview: Create a Kubebuilder-based, read-only operator that periodically scans Jobs in an explicit namespace allowlist and writes a bounded, diff-friendly status report into a CustomResource per namespace. Plan is structured as milestone targets that you check off one-by-one, with explicit decision logs and explain-back checkpoints to satisfy your strict-learning rules.
todos:
  - id: m0_docs_chat_context
    content: Create `docs/` chat context + decisions + progress checklist docs (no code).
    status: pending
  - id: m1_preconditions_linux_k3s
    content: Verify Go/kubectl/kubebuilder/make and confirm k3s cluster access.
    status: completed
  - id: m2_kubebuilder_init
    content: Run `kubebuilder init` with stable placeholders and inspect generated skeleton read-only.
    status: completed
  - id: m3_api_design_doc
    content: Write and log CRD/API spec (fields, bounds, status shape) and choose namespaced vs cluster-scoped report strategy.
    status: completed
  - id: m4_execution_model_decision
    content: Choose and log execution model (RequeueAfter controller vs single-ticker runnable).
    status: completed
  - id: m5_generate_api_controller
    content: Generate API + controller skeleton and run locally against k3s (`make run`).
    status: pending
  - id: m6_classification_rules
    content: Design conservative Job classification rules in docs, then implement and (optionally) unit test.
    status: pending
  - id: m7_scan_and_report
    content: Implement periodic scan + bounded status reporting end-to-end; validate idempotence and behavior with sample Jobs.
    status: pending
  - id: m8_in_cluster_deploy_k3s
    content: Deploy to k3s in-cluster; validate leader election and periodic scan behavior.
    status: pending
  - id: m9_org_cluster_validation
    content: Validate on org dev cluster, adjusting namespace allowlist and RBAC to real constraints.
    status: pending
  - id: m10_operational_polish
    content: Document config, failure modes, and logging expectations; keep within v1 non-goals.
    status: pending
isProject: false
---

# Operator-101 — Zombie & Stale Job Detection Operator (Detailed Target Plan)

## Working agreement (strict-learning compliance)

- You will write **100% of code and manifests**. I will only mentor: clarify concepts, interpret errors, challenge decisions, and review your reasoning.
- We will proceed one “layer” at a time (Go / Kubernetes behavior / controller design / debugging / architecture trade-offs).
- Every time there are options, you will **record a decision + what you lose** by rejecting alternatives (Rule 7).
- After each conceptual chunk, you will do an **explain-back** (Rule 8).

## Repo facts right now

- A Kubebuilder project skeleton **exists** (created via `kubebuilder init`).
- Key scaffold artifacts present: `go.mod`, `PROJECT`, `cmd/main.go`, `Makefile`, `config/*`.
- You already have context docs: [docs/00-session-handoff.md](docs/00-session-handoff.md) and [docs/01-problem-and-operator-context.md](docs/01-problem-and-operator-context.md).
- Primary dev target: **k3s**; secondary validation: org dev cluster.

## Learner state (so we focus intentionally)

- Minimum map achieved:
  - `PROJECT` is kubebuilder metadata used for future scaffolding/plugin behavior
  - `cmd/main.go` is the entrypoint that configures and starts the controller-runtime manager
  - `config/manager/manager.yaml` is the in-cluster deployment manifest
  - `config/default/kustomization.yaml` composes resources/patches into a deployable bundle
- Gaps to prioritize next:
  - Kustomize “assembly” mental model (resources vs patches)
  - Go basics needed to read controller-runtime code comfortably (`package`, imports, `init()` vs `main()`, module path/imports)
  - What “secure metrics/webhook/leader election” mean in the scaffold (at a conceptual level)

## Milestone 0 — “Chat context” + decision/progress logging (docs-only)

**Target**: Make the repo self-explanatory for future sessions.

- Create a single “chat context” doc in `docs/` that contains:
  - The operator intent + non-goals (point to/quote from `01-problem-and-operator-context.md`).
  - The strict-learning rules in 1 page (point to `.cursor/rules/*`).
  - A tiny “How to work with AI on this repo” section (one-layer questions, explain-back contract).
- Create `docs/DECISIONS.md` (or similar) where each decision entry contains:
  - **Decision**, **Alternatives**, **What we lose**, **Why chosen**, **Date**.
- Create `docs/PROGRESS.md` (checklist) mirroring the milestones below.

**Stop point**: Explain back why we’re writing docs *before* code.

## Milestone 1 — Preconditions (Linux + k3s)

**Target**: Tooling is verified; no guessing.

- Verify locally:
  - `go version` (≥ 1.21)
  - `kubectl version --client`
  - `kubebuilder version` (≥ 3.x)
  - `make --version`
- Verify cluster access to k3s:
  - `kubectl config current-context`
  - `kubectl get nodes`
  - Confirm you can create CRDs on k3s (we’ll test later during install).

**Failure mode handling** (in-scope for me): you paste the exact error output; I help debug.

**Stop point**: Explain back what `kubebuilder init` generates and why we freeze “zero business logic” first.

## Milestone 2 — Kubebuilder init (skeleton only)

**Target**: A clean Kubebuilder project skeleton exists; we do not touch generated code except by following Kubebuilder workflow.

- Pick a stable placeholder:
  - **Domain**: `ops.thakur.dev` (placeholder; stable)
  - **Repo**: `github.com/thakur/operator-101` (placeholder; stable)
- Run `kubebuilder init`.
- Inspect (read-only) these files to understand the shape:
  - `main.go`, `PROJECT`, `Makefile`, `config/default/*`, `config/manager/*`

**Decision log entry**: why this domain/repo is “stable enough” for learning.

**Stop point**: Explain back what the controller-manager “Manager” does at runtime.

## Milestone 3 — API design checkpoint (no code yet)

Your doc says:

- “One namespace-scoped CustomResource per namespace”
- “Status-only output”
- “Bounded and summarized findings”

**Target**: A written API spec (in docs) that you can defend.

### 3A. Choose CR strategy (record trade-offs)

You must choose one of these (and log it in `docs/DECISIONS.md`):

- **Option 1: Namespaced Report CR in each target namespace**
  - Gain: aligns with “per-namespace object”; least cross-namespace data coupling.
  - Lose: you must ensure a report object exists per namespace (manual creation or operator-created), and RBAC spans multiple namespaces.
- **Option 2: One cluster-scoped Report CR containing per-namespace sections**
  - Gain: single object to find; simpler “global ticker”.
  - Lose: cluster-scoped surface area; may be harder on restricted org clusters.

### 3B. Define the minimal CR spec/status fields (bounded)

Write in docs:

- **Spec**: scan interval, namespace allowlist reference (or “operator flags only”), thresholds.
- **Status** (bounded):
  - lastScanTime, scanDuration
  - summary counts (healthy / suspicious / zombie)
  - top-N list of findings (each with name, reason, age/duration)
  - conditions for “ReportReady” / “ScanFailed”

**Stop point**: Explain back what makes status “diff-friendly” and why bounded output matters.

## Milestone 4 — Controller execution model checkpoint (record trade-offs)

Your context doc states “single global ticker” and “periodic truth”. Kubebuilder defaults are reconcile-per-object.

**Target**: Choose the runtime model and log the trade-off.

- **Model A: Controller reconcile with `RequeueAfter**`
  - Gain: uses controller-runtime idioms; easier leader-election correctness.
  - Lose: periodic behavior is per-object (unless you centralize into one object); may drift from “single global ticker”.
- **Model B: Manager Runnable with one ticker**
  - Gain: exactly matches “single ticker + full recompute”.
  - Lose: more custom wiring; fewer guardrails from reconcile patterns.

**Stop point**: Explain back how leader election interacts with periodic scans.

## Milestone 5 — Implement the CRD + controller skeleton (first code)

**Target**: CRD exists, controller starts, but business logic is stubbed and safe.

- Generate API types and controller via Kubebuilder.
- Ensure RBAC is minimal:
  - list/watch/get Jobs in allowed namespaces
  - get/list/watch/update status for your Report CR
- Run locally against k3s using `make run` (preferred early path; avoids image build/push friction).

**Stop point** (challenge): What could go wrong if we accidentally broaden RBAC to “all namespaces”? What are you intentionally not handling?

## Milestone 6 — Job classification rules (design first, then code)

**Target**: Conservative, objective rules using only Job fields/status (no Pods/logs).
Write a short spec in docs first:

- Inputs allowed: Job `.spec` and `.status` (conditions, startTime, completionTime, failed/succeeded/active).
- Initial categories (example shape): Healthy / Suspicious / Zombie.
- Reasons list (finite): BackoffLimitExceeded, DeadlineExceeded, FailedLongAgo, RunningTooLong, NeverScheduled (if derivable), Suspended (if you include it), Unknown.
- Threshold sources: operator flags/env (v1), not per-namespace config.

Then you implement classifier code yourself.

**Verification targets**:

- You can point to one Job YAML and predict its category before running code.
- (Optional but high-signal) table-driven unit tests for classifier.

**Stop point**: Explain back why we avoid Pod inspection in v1 and what accuracy we give up.

## Milestone 7 — Scan loop + report writing (end-to-end)

**Target**: Periodic scan runs; status report updates correctly and stays bounded.

- Implement “list Jobs in namespace allowlist”
- Compute findings
- Write results to status
- Enforce hard bounds (top-N findings, max message sizes)

**Verification targets**:

- Create a few sample Jobs (successful, failing, long-running) and confirm report reflects them.
- Confirm re-runs are idempotent (same inputs → same status ordering/shape).

## Milestone 8 — Deploy to cluster (k3s first)

**Target**: Runs as a Deployment in-cluster.

- Produce manifests via `make deploy` workflow.
- Decide image strategy for k3s:
  - local build + import, or
  - push to registry your k3s can pull from.
- Validate leader election and periodic scan behavior in-cluster.

**Org cluster validation**: only after k3s is stable; adjust RBAC/namespace allowlist to org constraints.

## Milestone 9 — Operational polish (still within v1 non-goals)

**Target**: “Boring and correct” operational behavior.

- Clear logs (structured enough to debug; not the primary output)
- Config via flags/env, documented in `docs/`
- Failure modes documented:
  - API errors, partial namespace failures, timeouts
  - what status shows when a scan fails

## “Save progress” strategy (no forced git actions)

- After each milestone, update `docs/PROGRESS.md` and add a decision entry if any choice was made.
- If you want, you can ask me explicitly to help you craft commit boundaries/messages, but I will not commit unless you request it.

## Session rhythm (how we’ll mark targets one-by-one)

For each milestone we repeat:

- You attempt → paste errors/notes
- I debug/mentor → you fix
- You explain-back what you learned
- You update progress/decisions docs

