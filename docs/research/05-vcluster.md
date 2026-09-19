# 05 — vCluster / Loft Labs: the density and tenancy layer

**One-line summary:** not a bare-metal product at all, and that's the point. vCluster
solves the layer *above* us, and is a strong candidate to **embed** in our stack rather
than compete with.

---

## 1. The mechanism

A **virtual cluster** is a real Kubernetes control plane (k3s, k0s, or vanilla
apiserver+etcd, or apiserver+SQLite) running as **pods inside a namespace of a host
cluster**.

```
  HOST CLUSTER (real nodes, real kubelets, real CNI/CSI)
  ┌──────────────────────────────────────────────────────┐
  │  namespace: tenant-a                                 │
  │   ┌──────────────┐                                   │
  │   │ vcluster pod │  apiserver + etcd/sqlite + syncer │
  │   └──────┬───────┘                                   │
  │          │ syncer: rewrites & copies objects         │
  │          ▼                                           │
  │   pods: tenant-a-frontend-x-default-x-tenant-a  ◀────┼── actually scheduled
  │         tenant-a-api-x-default-x-tenant-a            │   by the HOST scheduler
  └──────────────────────────────────────────────────────┘
```

The **syncer** is the whole trick. Tenant creates a Pod in their virtual cluster →
the syncer creates a corresponding Pod in the host namespace with a mangled name →
the host schedules it → status syncs back. Some resources sync "down" (Pods, Services,
ConfigMaps, Secrets, PVCs, Ingress), some sync "up" (Nodes, StorageClasses, some
CRDs), most (Deployments, CRDs, RBAC, ServiceAccounts, operators) live **only in the
virtual cluster**.

**What the tenant gets:** full cluster-admin. Their own CRDs. Their own admission
webhooks. Their own API server version. Their own operators. `kubectl get nodes` works.
They cannot tell it isn't a real cluster.

**What the operator gets:** ~50–200MB and a few hundred millicores per "cluster",
created in seconds, deleted in seconds. Versus 3 control-plane nodes per real cluster.

---

## 2. The isolation caveat, and vNode

The classic critique: workloads still run on **shared host nodes, sharing a kernel**,
scheduled by the host scheduler. Namespace-level isolation, not cluster-level. Fine for
internal teams; questionable for hostile multi-tenancy (selling compute to strangers).

Mitigations in the product: dedicated node pools per vcluster (Kamaji-style), network
policies, gVisor/Kata runtimes, and — since 2025 — **vNode**, which adds
runtime/syscall-level isolation so a tenant gets "their own node" semantics on shared
hardware.

For a bare-metal GPU business, the practical answer is usually simpler: **GPU nodes are
allocated whole (or by MIG) to a tenant anyway**, so node-level sharing isn't the
concern; what you want from vCluster is the *control plane* density and the
self-service speed.

---

## 3. vCluster Platform (ex-Loft)

The commercial layer: UI, SSO, projects/teams, vcluster templates, quotas,
**sleep mode** (scale a vcluster to zero when idle, wake on first request — real money
saved on dev/test), cost reporting, audit, and multi-host-cluster management.

Adopted notably by GPU cloud providers who want "give every customer a cluster" without
buying every customer three control-plane servers.

---

## 4. Where it sits relative to the others

| Approach | Control plane | Nodes | Isolation | Cost/cluster | Good for |
|---|---|---|---|---|---|
| Real cluster | 3 dedicated nodes | dedicated | strongest | very high | prod, edge, disconnected |
| **HyperShift / Kamaji / k0smotron** | pods on hub | **dedicated** | strong | low | regional DC, IaaS, neoclouds |
| **vCluster** | pods on host | **shared** (or dedicated pools) | medium (→ strong with vNode) | ~zero | dev/test, internal teams, density |
| Namespace | shared | shared | weakest | zero | trusted single tenant |

**The key distinction for our design:** Kamaji/k0smotron give *hosted control planes
with real dedicated bare-metal workers* — that's the right primitive for selling metal.
vCluster gives *maximum density on shared workers* — the right primitive for internal
platform teams and dev/test tiers.

**A serious product offers both, as different "cluster classes" in the same catalog.**

---

## 5. Business model

Open source core (Apache 2), commercial platform for multi-tenancy/UI/SSO/sleep-mode.
Classic, healthy open-core. Worth studying as a **licensing template**: the OSS project
is genuinely useful standalone, the paid tier is everything a *company* (not a
developer) needs.

## 6. Where it hurts

- Syncer edge cases with operators/CRDs that assume cluster-scope or node access.
- Not for disconnected edge (the host cluster is the dependency).
- Nothing below L7. No metal, no OS, no network.
- Shared-kernel questions in hostile tenancy (partially addressed by vNode).

## 7. What we steal / use

- **Use it directly** as our "dev/test cluster class" and for internal platform tenancy.
- **Sleep mode** is a genuinely great FinOps feature worth replicating for GPU clusters
  (idle GPU node pools are the single biggest waste in AI infra).
- **The open-core split** (OSS = the engine; paid = tenancy, SSO, UI, cost, audit) is
  the right commercial template for us.
