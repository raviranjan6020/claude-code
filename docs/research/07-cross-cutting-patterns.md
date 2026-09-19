# 07 — Cross-Cutting Patterns: the actual design decisions

Strip away branding and every product in this space is a set of answers to the same
eight questions. These are the decisions *we* have to make. For each I've given the
options, the trade-off, who chose what, and a recommendation.

---

## D1. How does the hub talk to the spoke?

| Model | Mechanism | Who uses it | Gets you | Costs you |
|---|---|---|---|---|
| **Pull / reconcile** | Spoke agent polls hub API, applies desired state | OCM `ManifestWork`, Fleet `BundleDeployment`, Flux, Azure Arc | Best scale; no inbound; hub stores no spoke credentials; survives partition | No interactive ops; status lag |
| **Push with stored kubeconfig** | Hub holds creds, calls spoke API | Argo CD, plain scripts | Simple; immediate | Hub is a credential honeypot; needs network path; poor scale |
| **Reverse tunnel** | Spoke dials out, holds session; hub multiplexes through it | Rancher `remotedialer`, Rafay ZTKA, konnectivity | Interactive kubectl/exec/logs/port-forward, full audit, works behind NAT | Hub = traffic concentrator + SPOF for access; connection churn at scale |
| **Overlay VPN** | WireGuard mesh from every node to hub | **Sidero SideroLink**, Tailscale/Headscale, Netbird | Node-level reach (not just API), simple mental model, strong crypto | Key management; hub sees a huge mesh; MTU/NAT edge cases |

**Recommendation: all three, layered, with different jobs.**

```
  Desired state      → PULL      (spoke agent; scales to 10k; partition-tolerant)
  Interactive ops    → TUNNEL    (on-demand reverse tunnel; fully audited; short-lived)
  Node-level rescue  → WIREGUARD (optional, opt-in, for "the cluster is broken" cases)
  Telemetry          → PUSH-OUT  (spoke → hub remote_write; never hub-initiated)
```

The rescue path matters more than it sounds. When a spoke's control plane is dead, a
Kubernetes-API-based tunnel is useless — you need to reach the *node*. Sidero solved
this by making WireGuard the substrate and the Talos API the target. This is a real
differentiator for a support organisation.

**Identity**: never a shared token. Spoke registers with a one-time bootstrap token →
CSR → hub-issued short-lived client cert (OCM's model), auto-rotated. Consider SPIFFE/
SPIRE for workload identity if we want a single identity plane across fleet + tenants.

---

## D2. Where does the control plane physically live?

| | On-spoke (3 nodes) | On-spoke (SNO) | **Hosted on hub/regional** |
|---|---|---|---|
| Survives WAN loss | ✅ full | ✅ full | ❌ workloads run, but no API/scaling/self-heal |
| Hardware cost per site | 3 servers | 0 extra | 0 extra |
| Upgrade | rolling node reboots | risky, outage | **pod rollout, seconds, rollback** |
| etcd ops | per-site backup nightmare | same | centralised, `etcd-druid`-style |
| Density | 1 cluster : 3 servers | 1:1 | **100s of CPs on a few nodes** |
| Good for | regional DC, disconnected edge | far edge, retail | neoclouds, IaaS, dev/test, many-small-clusters |

Implementations: HyperShift (OpenShift), **Kamaji** (vanilla, most mature OSS),
**k0smotron** (k0s, CAPI-native), Gardener shoots, vCluster (shared workers).

**Recommendation:** make **cluster class** a first-class catalog concept, not an
architectural commitment:

- `edge-sno` — single node, local CP, local etcd, image-based upgrade
- `edge-ha` — 3 nodes, local CP, disconnected-tolerant
- `dc-standard` — 3+ nodes, local CP, full stack
- `hosted` — CP as pods in the regional tier, dedicated bare-metal workers (Kamaji)
- `virtual` — vCluster on shared workers (dev/test, internal tenants)

Same API, same GitOps, same addons. The customer picks per workload.

---

## D3. Two tiers or three?

**Two independent teams converged on three:** Mirantis (management → regional → child)
and SAP Gardener (garden → seed → shoot). Red Hat and Rancher are two-tier and both
strain at scale.

Why the middle tier exists:

- Provisioning is **L2-adjacent and bandwidth-heavy**. Ironic conductors, DHCP/PXE,
  and multi-GB OS image caches must be *near the metal*.
- Failure domains: a hub outage shouldn't stop a DC from replacing a failed node.
- Tunnel/agent fan-in: 40 regional connections to the hub beats 4,000 direct ones.
- Data residency/sovereignty: telemetry and secrets can stay in-region.
- Air gap: a regional tier can be fully disconnected while the global hub sees only
  periodic reconciliation.

**Recommendation: design for three tiers from day one, but ship the regional tier as
optional and collapsible** (a small deployment runs hub+region as one cluster). Adding
a tier later is a rewrite; collapsing one is a config flag.

```
  GLOBAL HUB            intent, identity, catalog, tenancy, billing, UI/API
       │                (small, HA, can be SaaS or customer-hosted)
       ▼
  REGIONAL / SITE       Ironic + DHCP/PXE + image cache + registry mirror
  CONTROLLER            + hosted control planes + local telemetry buffer
       │                (one per DC / per metro / per large edge site)
       ▼
  WORKLOAD CLUSTERS     the actual customer clusters
```

---

## D4. Mutable or immutable OS?

The industry has decided: **immutable, image-based, A/B, delivered as an OCI artifact.**

| | Mutable + config mgmt | Immutable A/B image |
|---|---|---|
| Examples | Ubuntu + Ansible, RHEL + Satellite, MAAS + curtin | Talos, Flatcar, SL Micro, Kairos, RHCOS, **bootc**, Ubuntu Core |
| Upgrade | in-place package churn | atomic swap + reboot, **instant rollback** |
| Drift | guaranteed, over time | impossible by construction |
| CVE response | patch N nodes, pray | rebuild image once, roll fleet |
| Provenance | "what's installed here?" is a question | image digest *is* the answer |
| Custom drivers | trivial | needs layering / system extensions |
| Legacy agents (backup, EDR, CMDB) | fine | **friction — this is the real adoption blocker** |

**The bootc convergence is the important insight.** With `bootc` (and Kairos, and
Flatcar's sysext, and Talos's Image Factory), the OS is built with a Containerfile,
stored in an OCI registry, signed with cosign, mirrored with `oras`/Harbor/Zot, and
pulled by the node. Which means:

> **One registry, one signing chain, one mirroring tool, one air-gap bundle format
> covers your OS, your Kubernetes components, your add-ons, and your workloads.**

That collapses four independent supply chains into one. For an air-gapped/sovereign
product this is worth an enormous amount, and it is *not* yet well productised by
anyone outside Red Hat.

**Recommendation:** bootc-based images as the default (Ubuntu/RHEL/Debian-derived, so
customers' EDR/backup agents still install), **Talos as an opt-in hardened class** for
customers who want maximum lockdown. Build and sign images centrally; mirror to
regional registries; nodes pull locally.

---

## D5. Cluster API, or our own?

**For CAPI:**
- A huge provider ecosystem we'd never build: vSphere, OpenStack, Proxmox, Nutanix,
  KubeVirt, Harvester, Metal3, MAAS, Tinkerbell, every cloud.
- Bootstrap/CP providers for RKE2, k3s, k0s, Talos, kubeadm, microk8s.
- `ClusterClass` + managed topologies + variables + patches = templating that exists.
- Runtime Extensions / lifecycle hooks let us inject our logic at defined points.
- Hiring, docs, community, and "is it standard?" in RFPs.
- Chosen by: Spectro, k0rdent, SUSE Edge/Turtles, EKS-A, Kubermatic-adjacent, Rancher v2.

**Against CAPI:**
- Genuinely complex; many controllers, many CRDs, subtle reconcile ordering.
- Bare-metal providers are the least polished part of the ecosystem.
- Hard to give good error messages through five layers of abstraction — the #1 UX
  complaint. ("Why is my cluster stuck?" → read six controllers' logs.)
- Sidero rejected it outright, and their UX is visibly better as a result.
- You inherit upstream's release cadence and breaking changes.

**Recommendation: CAPI as the *engine*, never as the *API*.**

Expose our own opinionated CRDs (`Site`, `MachinePool`, `ClusterSpec`, or k0rdent-style
`ClusterTemplate`/`ClusterDeployment`) and compile them down to CAPI objects. Users and
our UI never see a `Metal3MachineTemplate`. Crucially, **own the status/condition
aggregation**: a single `Reason` + human-readable `Message` on our CR that explains
what's actually blocked, synthesized from the underlying CAPI/Metal3/Ironic conditions.
That translation layer is a real product feature and the thing k0rdent is missing.

---

## D6. How do 1,000 sites get expressed without 1,000 YAML files?

Every vendor's answer:

| Vendor | Mechanism |
|---|---|
| Red Hat | `ClusterInstance` (site facts) + installation templates (method) + `PolicyGenerator` (config) |
| Rancher | Fleet `Bundle` + per-target `helm.values`/kustomize overlays |
| Spectro | Cluster Profile (layered, versioned) + cluster-level overrides |
| k0rdent | `ClusterTemplate` + `ClusterDeployment` values + `MultiClusterService` |
| CAPI | `ClusterClass` + variables + JSON patches |
| Sveltos | `ClusterProfile` + label selectors + templating from cluster resources |

The shape they all converge on:

```
  TEMPLATE (what a site of this type looks like)   ← few, versioned, reviewed
       ×
  SITE FACTS (IPs, VLANs, MACs, BMC creds, geo)    ← many, generated, not handwritten
       ×
  PROFILE OVERLAY (this tier/region/tenant differs) ← a handful
       =
  RENDERED DESIRED STATE
```

**The part everyone under-invests in is "site facts".** Hand-writing 1,000 site YAMLs
is the actual failure mode in real ZTP deployments. Site facts must come from a real
**source of truth with an API** — NetBox/Nautobot, a CMDB, or our own inventory — and
be *generated*. This is a place we can be meaningfully better: ship the inventory/IPAM
model as part of the product with an import path from NetBox and from a spreadsheet,
because the customer's reality is a spreadsheet.

---

## D7. What does day-2 rollout look like?

Everyone can `kubectl apply`. Almost nobody can safely upgrade 500 sites.

The primitives that matter (Red Hat's TALM is the reference implementation):

1. **Selection** — which clusters, by label/set/query.
2. **Batching** — N at a time, with explicit canaries first.
3. **Pre-caching** — pull images/OS artifacts to the site *before* the window opens.
   Critical on thin WAN links; the difference between a 5-minute and 5-hour outage.
4. **Maintenance windows** — per-site timezone-aware.
5. **Pre-flight gates** — disk space, backup taken, health green, no active alerts.
6. **Backup before mutate** — etcd snapshot, OS image pinned for rollback.
7. **Verification** — post-condition checks, not just "the apply succeeded".
8. **Abort/rollback policy** — stop the whole campaign after K failures.
9. **Resumability** — a campaign must survive a hub restart and a 3-day pause.

**Recommendation:** build a `RolloutCampaign` CRD with exactly these semantics, distro-
agnostic, covering OS + Kubernetes + addons + firmware. **This is the single feature
most likely to win a deal against Rancher/Spectro**, because it's what operations teams
actually lose sleep over and only Red Hat has it.

---

## D8. Air gap

Treat this as a **build artifact**, not documentation.

What must cross the gap: container images (platform + workloads), Helm charts, OS
images, firmware blobs, OS packages, CAPI provider manifests, signatures/attestations,
SBOMs, CVE feeds, and the product itself.

Design:
- Everything is an **OCI artifact** (`oras` for non-image artifacts). One format.
- A `bundle` CLI produces a single signed tarball for a given release + selected
  extras, with a manifest and checksums.
- A regional **registry mirror** (Zot — tiny, OCI-native, Apache 2 — or Harbor when the
  customer wants RBAC/replication/scanning).
- Verification: cosign policy enforced *in-cluster* so an unsigned artifact can't run,
  and that verification must work offline (bundled trust roots, no Rekor round-trip).
- Don't forget: **time sync** (NTP/PTP) and **certificate bootstrapping** — air-gapped
  sites with wrong clocks fail TLS in confusing ways, and it's a top support call.

---

## D9. Scale limits — what actually breaks

Useful to know before we pick numbers for the pitch deck:

| Bottleneck | Typical wall | Mitigation |
|---|---|---|
| Hub etcd object count | ~10–20k clusters × N objects each | regional tier; don't mirror spoke objects to the hub |
| Reverse tunnel fan-in | ~1–2k persistent sessions/instance | regional relays; on-demand rather than persistent tunnels |
| Argo CD Application count | ~2–5k Applications/instance | Fleet/Sveltos model instead; or shard Argo per region |
| Git repo size / reconcile time | 1000s of site dirs | generated manifests + OCI artifacts, not a monorepo of YAML |
| Metrics cardinality & egress | the #1 surprise cost | aggregate at the regional tier; strict allowlists (RHACM does this); downsample before WAN |
| Certificate churn | thousands of short-lived certs | automated rotation; test the 400-days-offline case |
| Image distribution | GB × N sites over WAN | regional mirror + P2P (Dragonfly/Spegel) + pre-caching |

---

## D10. Where the business models differ

| Vendor | Unit | Rough position | Implication |
|---|---|---|---|
| Red Hat | core-pair/socket | highest | prices itself out of edge & GPU volume |
| SUSE/Rancher | per node | low–mid, all OSS | can't capture value from big deployments |
| Mirantis | per node + services | mid, services-heavy | scales with headcount, not software |
| Rafay | per cluster/node/GPU, SaaS | mid–high | fastest revenue, but closed |
| Spectro | per node/cluster, SaaS | mid–high | edge volume pricing is a real fight |
| Sidero | per node/cluster | low–mid | small TAM by choice |
| **vCluster** | **per virtual cluster** | open core | cleanest OSS→paid split |

**The open-core split that works** (vCluster, Grafana, HashiCorp-pre-BSL): the OSS
thing is genuinely, completely useful for one team/one cluster/one site. The paid thing
is everything a *company* needs: multi-tenancy, SSO/SAML, audit, air-gap bundles, FIPS
builds, rollout campaigns, metering/chargeback, support SLA, and the compliance
paperwork. Never cripple the engine; charge for the organisation.
