## Operator-101 — Session Handoff (Context Window)

Date: 2026-02-10

### Session goal (today)

Get a **running kubebuilder project skeleton** with **zero business logic**.
No CRDs, no controllers, no watches, no reconcile logic yet.

### Update (Linux continuation) — what actually happened

We continued on **Linux (Ubuntu 24.04.3 LTS)** with **k3s** as the primary dev cluster.

#### Preconditions verified (Linux)

- **kubectl**: client works (v1.34.3)
- **k3s cluster**: reachable
  - context: `default`
  - nodes: `kubectl get nodes` returned a Ready node (k3s v1.33.6+k3s1)
- **make**: installed (GNU Make 4.3)
- **kubebuilder**: installed and working
  - path: `/usr/local/bin/kubebuilder`
  - version: v4.11.1
- **go toolchain**:
  - apt provided `go1.22.2` at `/usr/bin/go`
  - kubebuilder v4.11.1 required **Go >= 1.23** for the v4 plugin, so we installed Go via **official tarball**
  - tarball Go installed at `/usr/local/go/bin/go` (Go `1.25.7`)
  - IMPORTANT: PATH must prefer `/usr/local/go/bin` over `/usr/bin` (otherwise Go falls back to 1.22.2)

#### The actual blocker we hit (and resolved)

- `kubebuilder init` failed initially with:
  - “go version go1.22.2 is incompatible: plugin requires go1.23 <= version < go2.0alpha1”
- Resolution:
  - installed Go tarball under `/usr/local/go`
  - exported PATH for the session (`/usr/local/go/bin` first)
  - verified with `which go` and `go version` before retrying

#### Milestone 2 result: kubebuilder init succeeded (repo root)

Command used (from repo root):

- `kubebuilder init --domain ops.thakur.dev --repo github.com/menotthakur/k8s-zombie-workload-operator`

Notes:
- kubebuilder warned “target directory is not empty” (risk: overwrites/conflicts). It still completed successfully.
- Generated project skeleton now exists in this repo, including:
  - `go.mod`, `go.sum`
  - `PROJECT`
  - `cmd/main.go`
  - `Makefile`
  - `config/*` (kustomize manifests, RBAC, manager deployment)
  - CI/dev tooling (`.github/workflows/*`, `.golangci.yml`, etc.)

#### Current learning map (minimum mental model achieved)

What is understood at a usable “map” level:
- `PROJECT`: kubebuilder metadata (CLI version, layout/plugin, domain, repo) used by kubebuilder tooling later.
- `cmd/main.go`: Go entrypoint (like `main.cpp`), boots controller-runtime Manager and config (metrics/probes/leader election).
- `config/manager/manager.yaml`: the in-cluster deployment manifest for controller-manager.
- `config/default/kustomization.yaml`: “assembly” layer that composes resources + patches into a deployable bundle.

#### Known gaps (to focus next, without derailing v1)

- Kustomize mental model: resources vs patches vs overlays (how `config/default` produces final manifests).
- Go basics needed for reading controller code confidently:
  - `package main`, imports, `init()` vs `main()`, modules/import paths
- Controller-runtime concepts: what Manager provides, what secure metrics/webhook wiring is doing in the scaffold.

#### Explicit v1 scope decision (recorded)

- v1 will be **Jobs-only** (no Pod inspection, no logs/events). This is an intentional trade-off for safety and simplicity.

### Where we stopped

We stopped at **STEP 1 — Preconditions** (verify tools, don’t assume).
Reason: Windows environment/tooling friction blocked `go` and `kubebuilder`.

### What we verified (facts)

- **Workspace**: `e:\Solytics\Tasks\Operator-101`
- **Kubernetes client**: `kubectl version --client` worked (v1.30.0).
- **Kubebuilder**: `kubebuilder version` failed with `command not found` (not installed / not on PATH).
- **Go**: `go version` and `go env` failed due to toolchain selection error.

### The concrete blocker (copyable evidence)

- `go env GOTOOLCHAIN` returned: `go1.21`
- Then `go env GOPATH GOROOT` / `go version` failed with:
  - `invalid toolchain: go1.21 is a language version but not a toolchain version (go1.21.x)`

Meaning (understood, not memorized):
- `go1.21` is a **language line** (major.minor).
- `GOTOOLCHAIN` requires either:
  - a **patch toolchain** (e.g., `go1.21.x`), or
  - a **mode** like `auto`.

### What we tried / changed (Windows)

- Confirmed `go` binary resolved to: `/c/Program Files/Go/bin/go` (Git Bash path view).
- Set a Windows user environment variable: `GOTOOLCHAIN=auto` (intended fix).
- Reminder: env var changes require a **new terminal** to take effect.

### Key decision: move setup to Linux (why)

We decided to do the setup on **Linux (dual boot)** to minimize “Windows tax”:
- Kubebuilder + make + container/K8s tooling are most reliable on Linux.
- Less time on PATH/env/make quirks, more time on operator design learning.

Trade-off accepted:
- Re-do tooling setup on Linux, but lower ongoing friction.

### What NOT to do yet (guardrails)

Do not create APIs/CRDs/controllers yet.
Do not add any operator logic.
Do not edit generated files until the skeleton is created and observed.

### Next steps (Linux) — do only these first

Run and verify these on Linux:

- `go version` (expect Go ≥ 1.21)
- `kubectl version --client`
- `kubebuilder version` (expect kubebuilder ≥ 3.x)

Do not proceed until all three work.

### After preconditions pass (the next irreversible step)

From repo root, run `kubebuilder init` with a stable fake domain + repo:
- Domain: pick something like `ops.<yourname>.dev` (stable; don’t overthink)
- Repo: `github.com/<yourname>/k8s-zombie-workload-operator` (or similar)

Then freeze and observe:
- Run `tree -L 2`
- Open read-only:
  - `main.go`
  - `Makefile`
  - `config/default/kustomization.yaml`

### Stop-point contract (how to continue)

After `kubebuilder init` is done, stop and respond with only one:

1) “Kubebuilder init done, here’s what confused me”
2) “Kubebuilder init done, explain manager & main.go deeply”
3) “Kubebuilder init done, ready to define CRD (design only, no code)”

