# k8s-zombie-workload-operator
A Kubernetes operator periodically scans a configured set of namespaces, audits Jobs for zombie conditions, and reports findings in a namespace-scoped CustomResource status. It is read-only and does not mutate workloads.
