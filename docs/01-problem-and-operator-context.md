# Kubernetes Zombie & Stale Workload Detection Operator

## Context, Intent, and Execution Plan

---

## 1. Background & Real-World Context

Modern Kubernetes clusters, especially in startups and fast-moving teams, accumulate abandoned batch workloads over time.

These workloads typically arise from:

- Failed data pipelines
- Experimental batch jobs
- Retries that exhausted silently
- CI/CD or ad-hoc Jobs forgotten after failure
- Human debugging sessions that never cleaned up resources

While Kubernetes exposes raw status, it does not provide semantic classification such as:

- "this Job is abandoned"
- "this Job will never succeed"
- "this Job is no longer useful"

As a result:

- Namespaces become cluttered
- Engineers lose confidence in `kubectl get jobs`
- Real failures are hidden among old noise
- Operational cognition cost increases

This problem already exists in real clusters and worsens with time.

---

## 2. Core Problem Statement

There is no reliable, structured, namespace-level mechanism in Kubernetes to continuously identify and summarize zombie or abandoned batch workloads in a safe, non-intrusive way.

Key gaps:

- Kubernetes does not define "zombie" workloads
- Humans rely on manual inspection
- Scripts are ad-hoc and non-authoritative
- There is no single source of truth for workload health

---

## 3. Why This Operator Exists

This operator exists to answer one simple but powerful question:

> "What batch workloads in this namespace are no longer healthy or meaningful right now?"

It does this by:

- Periodically inspecting Jobs
- Applying conservative, objective rules
- Producing a structured health report
- **Without mutating any workload**

This is **visibility, not remediation**.

---

## 4. Explicit Non-Goals (Very Important)

This operator intentionally does **NOT**:

- Delete Jobs
- Restart Jobs
- Retry Jobs
- Emit alerts
- Send notifications
- Expose Prometheus metrics
- Inspect Pods deeply
- Handle CronJobs (v1)
- Enforce SLAs
- "Fix" anything

**Why:**

- Mutation destroys trust
- Alerts require policy decisions
- Remediation requires human intent
- v1 must be safe, boring, and correct

Any feature that increases blast radius is out of scope.

---

## 5. Operator Philosophy

This operator follows these principles:

- **Read-only** by design
- **Stateless** execution
- **Periodic truth** over event-based reactions
- **Conservative** classification
- **Structured output** over logs
- **Safety** over cleverness

If a design decision violates one of these, it is rejected.

---

## 6. Scope of Operation

### 6.1 Namespace Scope

- Operator scans a statically configured list of namespaces
- Namespaces are provided via operator configuration (env / flags)
- No dynamic namespace discovery in v1

**Rationale:**

- Predictable behavior
- Explicit ownership
- Zero surprise scans

### 6.2 Workload Scope

- Only Kubernetes Job resources
- Only controller-authoritative fields
- No Pod log or container inspection

---

## 7. Execution Model

### 7.1 Periodic Model

- Single global ticker
- Fixed scan interval
- Full recomputation every cycle

**Why:**

- Zombie detection is time-based
- Event-driven models miss slow failures
- Periodic scans are crash-safe and simple

### 7.2 Statelessness

- No in-memory history
- No PVCs
- No external storage
- Operator restart = clean slate

Kubernetes API is the source of truth.

---

## 8. Zombie Job — Conceptual Definition

A Job is considered **Zombie** if it satisfies any of the following high-level conditions:

- Kubernetes has exhausted retries and given up
- The Job has been running abnormally long without completion
- The Job failed long ago and was never acted upon

The operator **does not guess intent**.
It only interprets **observable state**.

---

## 9. Safety Guarantees

This operator guarantees:

- It will **never** mutate workloads
- It will **never** delete resources
- It will **never** block application execution
- It will **never** require persistent storage
- Operator crashes are harmless
- Re-running scans is always safe

These guarantees are **non-negotiable**.

---

## 10. Output Model (High-Level)

The operator produces:

- One namespace-scoped CustomResource per namespace
- Status-only output
- Bounded and summarized findings

The CustomResource status is:

- The single source of truth
- Human-readable
- Machine-consumable
- Diff-friendly

No output is hidden in logs.

---

## 11. Learning Intent (Why This Project Exists for Me)

This project is **not** about:

- Learning Go syntax in isolation
- Completing a tutorial
- Shipping a feature quickly

This project **is** about:

- Learning how real Kubernetes operators are designed
- Understanding controller trade-offs
- Thinking in reconciliation and state
- Building something production-shaped
- Becoming confident reading large Go infra codebases

**Depth > speed.**
**Reasoning > output.**

---

## 12. Boundaries for Future Expansion

The following are explicitly deferred to future versions:

- CronJob awareness
- Configurable per-namespace thresholds
- Metrics & alerting
- Automated cleanup
- Pod-level diagnostics
- Event-driven reconciliation

These are intentionally postponed to protect correctness.

---

## 13. Success Criteria

This project is considered successful if:

- The operator runs safely in a real cluster
- Zombie Jobs are correctly identified
- Status output is clear and bounded
- Design decisions are defensible
- I can explain every part without AI
- I can rebuild it again from memory

Anything else is secondary.

---

## 14. Final Commitment

This operator will remain:

- **Boring**
- **Conservative**
- **Safe**
- **Explicit**

Because boring infrastructure is good infrastructure.

---

*End of Context & Plan*
