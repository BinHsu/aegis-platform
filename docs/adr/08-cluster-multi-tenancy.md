# ADR-08: Cluster multi-tenancy — shared by default, dedicated by exception via a platform module

## Status

Proposed.

## Context

[ADR-07](07-workload-self-ownership.md) lets a workload onboard itself — its
own `Application` CR, its own IAM CRDs, no PR to the platform repo. That
answers *what a workload owns*. It does not answer *where a workload runs*.
The unstated default in this repo so far has been "share the per-region EKS
cluster `aegis-platform-${region}`", but the default has never been written
down, and the escape hatch — what to do when a workload genuinely cannot
share — has never been named.

Two postures are easy to slide into without an explicit decision:

- *Cluster per service* — every workload gets its own EKS cluster. Strongest
  isolation; operationally untenable at N > 2 for a small team.
- *Self-service dedicated clusters with no template* — workloads that want
  their own cluster spin one up via their own Terraform. Sounds like
  autonomy; produces baseline drift (different K8s versions, missing
  policies, ad-hoc Karpenter configs) within one release.

The right answer is neither — a default with a named, paved escape hatch.

## Decision

**Default: workloads share the per-region cluster.** Isolation is per
namespace, not per cluster:

- **Namespace** per workload, named with the full repo prefix per Rule 11
  (`aegis-core`, `aegis-greeter`, …) — no bare `aegis-` namespace.
- **NetworkPolicy** default-deny on the namespace, with explicit allows for
  the traffic the workload actually needs.
- **RBAC** — ServiceAccount + Role/RoleBinding scoped to the namespace.
- **ResourceQuota + LimitRange** on the namespace, so one workload cannot
  starve another.
- **Kyverno ClusterPolicies** enforce the platform floor — PodSecurity
  Standard `restricted` (per [ADR-06](06-security-and-runtime.md)), no
  `hostPath`, no privileged pods, no PVC where stateless workloads claim
  none.

**First escape hatch — dedicated nodes, shared control plane.** When a
workload needs an instance type the default Karpenter NodePool does not
serve (GPU, ARM, a memory-optimised family), or its appetite is large
enough to risk starving others, the workload's deploy repo declares a
Karpenter `NodePool` + `EC2NodeClass`, and pods bind to it via
NodeSelector and Taint+Toleration. Same EKS control plane, dedicated EC2.
This covers the vast majority of "I need my own cluster" cases — which
usually mean "I need dedicated nodes".

**Second escape hatch — dedicated cluster via `modules/dedicated-cluster/`.**
When the control plane itself has to be separate (different K8s minor
version, compliance boundary, different VPC/CIDR topology), the platform
repo exposes a `modules/dedicated-cluster/` Terraform module. The workload's
deploy repo invokes the module from its own `terraform/cluster/main.tf`,
assuming a deploy-time role into the workload account. The module enforces
the platform baseline: PSS `restricted`, Karpenter, ACK, Kyverno, Alloy, the
ALB controller, cert-manager. The escape hatch is paved — explicit,
documented, audited via the module's own PR review process and semver
upgrades.

## Decision criteria

Stay on the shared cluster unless **one** of these is true:

| Trigger | Promote to |
|---|---|
| Instance type / GPU not on the default Karpenter NodePool | Dedicated NodePool |
| Workload's steady-state appetite ≥ ~30% of the cluster (starvation risk) | Dedicated NodePool |
| Hard latency SLO that cohabitation breaks | Dedicated NodePool |
| Different K8s minor version (legacy stuck on n-2) | Dedicated cluster |
| Compliance boundary forbids a shared control plane (PCI / HIPAA / FedRAMP) | Dedicated cluster |
| Different VPC topology required (private CIDR, peering layout) | Dedicated cluster |

A workload that promotes to a dedicated NodePool stays inside this repo's
ArgoCD and observability wiring. A workload that promotes to a dedicated
cluster runs its own `modules/dedicated-cluster/` apply, but inherits the
same baseline.

## Considered alternatives

- **Cluster per service.** Rejected on operational cost. At five workloads
  across two regions that is ten clusters to patch, upgrade, scan, and
  observe. The control-plane overhead does not buy isolation the namespace
  + NetworkPolicy + Kyverno trio cannot give.
- **Cluster per team or domain.** Rejected on premature axis-split. This
  project has one operator; team boundaries are not real yet. Adding the
  axis now bakes in a structure the org does not have. Defer until team
  shape emerges and the cost of *not* splitting is visible.
- **App teams self-provision dedicated clusters via their own Terraform,
  without a platform module.** Rejected on baseline drift. Without the
  paved module, each dedicated cluster diverges — different K8s versions,
  missing Kyverno policies, ad-hoc Karpenter configs, missing Alloy
  wiring. The module is what makes the escape hatch safe.

## Consequences

- Default onboarding is namespace-only: no new AWS resources, no new
  cluster, no new control plane. The unit cost of the second, third, and
  fourth workload is a Kustomize overlay and a tagged deploy repo.
- The Karpenter NodePool tier absorbs the 90% case that gets misdiagnosed
  as "I need my own cluster" — almost always actually "I need dedicated
  nodes".
- `modules/dedicated-cluster/` is a paved-road artefact. Workloads that
  genuinely need a dedicated control plane do not get a wild-west DIY
  experience; they consume a hardened recipe and inherit baseline upgrades
  via module-version bumps.
- The platform team becomes a module author, not an infrastructure ticket
  queue. The module is the contract; the contract is versioned.
- Trade-off: shared-cluster failures are blast-radius-wide within a region.
  Multi-AZ replicas absorb pod / node / AZ failures; region loss is the
  cold-rebuild RTO path ([ADR-05](05-disaster-recovery.md)). A workload
  with a stricter blast-radius requirement promotes itself to a dedicated
  cluster — that is exactly what the escape hatch is for.
