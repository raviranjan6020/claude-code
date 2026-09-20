# 13 — Ingot: the P2 Build Plan

The executable plan. Supersedes the milestone table in doc 12 (see D-5).

**Goal:** a working platform-layer product — clusters created, addons applied, scaled,
upgraded, continuously reconciled, deleted — built on VMs, in 10 weeks, deliberately
unoriginal, so that P1 (the differentiated metal + firmware work) starts from a
platform rather than from nothing.

---

## 1. The done test

One sentence, no judgement calls:

> From a clean hub: `ingot cluster create prod-01 --template dc-standard` provisions a
> 3-node RKE2 cluster on EC2 with Cilium and the base addon set; a second cluster is
> formed from **pre-existing VMs** that registered themselves outbound; both can be
> scaled and upgraded; hand-editing a managed resource is reverted automatically; and
> `ingot cluster delete` removes everything. **Unattended, reproducible from scratch,
> recorded on video.**

When that video exists, P2 is done and P1 starts. Not when it is polished. Not when
anyone joins.

---

## 2. Architecture

```
╔════════════════════════════════════════════════════════════════════════╗
║  HUB — single k3s node   (--tier=global,regional,site : all collapsed) ║
╠════════════════════════════════════════════════════════════════════════╣
║   ┌─ OURS ─────────────────────────┐  ┌─ OSS ───────────────────────┐  ║
║   │ ingot-server                   │  │ Cluster API core            │  ║
║   │   CRDs: Site · Machine         │  │ CAPA        (EC2)           │  ║
║   │         ClusterTemplate        │→ │ CAPRKE2     (bootstrap+CP)  │  ║
║   │         Cluster                │  │ ingot-infra (pre-provisioned│  ║
║   │         AddonTemplate          │  │              hosts)  ★OURS  │  ║
║   │         AddonOverlay           │  │                             │  ║
║   │   renders → CAPI + Sveltos     │  │ Sveltos  (addon delivery)   │  ║
║   │   aggregates status → ONE      │  │ Flux     (git/OCI source)   │  ║
║   │     human-readable message     │  │ cert-manager                │  ║
║   │   registration: token→CSR→cert │  │                             │  ║
║   └────────────────────────────────┘  └─────────────────────────────┘  ║
║              ▲ ingot CLI / kubectl                                     ║
╚══════════════╪═════════════════════════════════════════════════════════╝
               │  spoke-initiated only (outbound TLS 443)
      ┌────────┴─────────┐                  ┌──────────────────────────┐
      │ PATH A — BYOH    │                  │ PATH B — CAPA            │
      │ pre-existing VMs │                  │ EC2, created on demand   │
      │ ingot-agent      │                  │ destroyed after each test│
      │  register        │                  │                          │
      │  inventory       │                  │ 3 × t3.medium            │
      │  exec bootstrap  │                  │ ap-south-1               │
      │ RKE2 + Cilium    │                  │ RKE2 + Cilium            │
      └──────────────────┘                  └──────────────────────────┘
       ↑ becomes the bare-metal path in P1     ↑ proves the abstraction
```

**Two provisioning paths on purpose.** Path B proves the model isn't a thin wrapper
around one provider. Path A — claiming machines that already exist — is structurally
identical to claiming bare-metal hosts, so it carries straight into P1 unchanged.

---

## 3. What we write vs what we consume

| # | Step | Whose code |
|---|---|---|
| 1 | Host runs `ingot join --token=…`, dials hub, CSR, gets cert | **OURS** — agent + registration controller |
| 2 | Agent reports inventory (CPU/RAM/disk/NIC/OS, **+ BIOS attrs read-only**) | **OURS** |
| 3 | `Machine` appears on hub, state `Available` | **OURS** |
| 4 | User declares a `Cluster` from a `ClusterTemplate` | **OURS** — the product surface |
| 5 | Controller renders CAPI + Sveltos objects | ⭐ **OURS** — the core of P2 |
| 6 | Machines claimed and bootstrapped | ⭐ **OURS** — `ingot-infra` provider |
| 7 | RKE2 config generated and executed | OSS CAPRKE2 + our agent |
| 8 | Cluster up, kubeconfig in a hub secret | OSS CAPI |
| 9 | Addons land | OSS Sveltos + Flux; **OURS**: the CRDs that compile to them |
| 10 | Continuous reconcile + drift correction | OSS Sveltos |
| 11 | One human-readable status line explaining what is blocked | ⭐ **OURS** |
| 12 | Scale / upgrade | OSS CAPI, orchestrated by **OURS** |

**Our code is ~15–20% of the running bytes and 100% of the product surface.**

### The one interesting piece: `ingot-infra`

A CAPI infrastructure provider for hosts that already exist. It is small **because CAPI
infra providers are only complex when they must _create_ machines — ours only has to
_claim_ them.** No cloud API, no quotas, no instance types, no IAM.

```
IngotCluster  → set ControlPlaneEndpoint, mark Ready
IngotMachine  → claim an Available Machine matching the selector
                hand bootstrap data to its agent
                wait for the Node, set ProviderID, mark Ready
```

~600–900 lines. Keeps **one** machine-lifecycle model instead of two, and it is the
same provider that claims bare-metal hosts in P1.

---

## 4. Scope

### In
- `Site` · `Machine` · `ClusterTemplate` · `Cluster` · `AddonTemplate` · `AddonOverlay`
- Metal-free: CAPA for EC2, `ingot-infra` for pre-existing VMs
- **One** distro: RKE2. **One** CNI: Cilium. MetalLB on the BYOH cluster only
- Three addons max beyond CNI: cert-manager, metrics-server, a monitoring agent
- Scale (`MachineDeployment` replicas), upgrade (happy path), drift reconciliation
- Single tenant, stock Kubernetes RBAC, CLI + `kubectl` only
- **Read-only BIOS attribute and firmware-version capture during inventory** — the one
  deliberate exception, so the P1 quirks dataset starts accumulating in week 4

### Explicitly NOT in
❌ A second distro (k3s, Talos, k0s) · ❌ a second CNI · ❌ OCM / fleet hub ·
❌ audited kubectl tunnel · ❌ `RolloutCampaign` · ❌ multi-tenancy, projects, quotas ·
❌ web UI · ❌ air-gap · ❌ hosted control planes / Kamaji · ❌ vCluster · ❌ GPU ·
❌ KubeVirt · ❌ network fabric · ❌ firmware *remediation* (capture only)

**And the one that will hurt: ❌ upgrade hardening.** Happy-path rolling upgrade is in.
Rollback, etcd quorum loss mid-upgrade, addon compatibility matrices, air-gap pre-pull,
maintenance windows and canaries are out. When an upgrade edge case appears in week 7,
**write it down and move on** — that is where Rancher's decade went.

---

## 5. Phases

| Phase | Weeks | Work | Done when |
|---|---|---|---|
| **0** | 1 | Monorepo, Go module, kubebuilder init, Makefile, CI, hub k3s. Install CAPI + CAPA **by hand** and read what it creates | `clusterctl` creates an EC2 cluster manually |
| **1** | 2–3 | `ClusterTemplate` + `Cluster` CRDs; controller renders CAPI objects; status/condition aggregation | `ingot cluster create` → RKE2 on EC2 |
| **2** | 4 | Sveltos + Flux; `AddonTemplate`/`AddonOverlay`; Cilium, cert-manager, metrics-server. **BIOS/firmware capture in inventory** | addons land automatically on create |
| **3** | 5–6 | `ingot-agent` + registration + `ingot-infra` provider | a NAT'd VM registers and joins a cluster unattended |
| **4** | 7 | Scale + upgrade (happy path only) | `ingot cluster scale` / `upgrade` work |
| **5** | 8 | Drift detection, periodic resync, condition reporting | hand-edit a managed resource → it reverts |
| **6** | 9–10 | CLI polish, one-command hub install, docs, **record the demo** | the video exists |

**Phase 0 deliberately automates nothing.** Watching `clusterctl` create a cluster by
hand, and reading every object it produces, is what makes phases 1–3 fast instead of
guesswork.

---

## 6. Infrastructure and cost

```
ALWAYS ON                       EPHEMERAL (per test, destroyed after)
┌────────────────────┐          ┌──────────────────────────────┐
│ HUB                │─creates─▶│ CAPA cluster                 │
│ k3s + CAPI + CAPA  │          │ 3 × t3.medium, ap-south-1    │
│ + CAPRKE2 + Sveltos│          │ ~2 hours per test            │
│ + ingot-server     │          └──────────────────────────────┘
└────────────────────┘
        │              ┌──────────────────────────────┐
        └─registers───▶│ 1–2 standing VMs = BYOH path │
                       │ small; stopped when idle     │
                       └──────────────────────────────┘
```

**Hub placement — prefer the already-paid-for rented server** (8GB+ RAM, 4+ cores). The
hub only makes *outbound* calls, so NAT is fine. Otherwise a t3.large EC2 instance;
t3.medium (4GB) is workable but tight with ~15 controller pods.

**Cost per test run:** 3 × t3.medium in ap-south-1 ≈ **₹4/hour total**. A two-hour test
is under ₹10.

**The trap:** forgetting teardown. Three nodes left running for a month is ~₹8,000.
Mitigations: an AWS Budget alert at ₹1,000 on day one, and teardown as the CLI default.

**Prerequisites:** IAM for CAPA (`clusterawsadm bootstrap iam create-cloudformation-stack`),
an SSH keypair in ap-south-1, a VPC (default is fine), and the budget alert.

---

## 7. Conventions

```
Product      Ingot
Repo         github.com/<org>/ingot            monorepo — not three repos
API group    ingot.sh/v1alpha1
Binaries     ingot           CLI
             ingot-server    --tier=global|regional|site  (comma-separated to collapse)
             ingot-agent     register · inventory · exec bootstrap · (later) tunnel
Tiers        global · regional · site          deployment modes, not products
```

Tier naming rejected: **GCM/RCM/LCM**. `LCM` already means *lifecycle management*
universally in this industry (Mirantis ships an "LCM controller"), and Rancher's
"local cluster" means the *management* cluster — so "local" for a spoke inverts the
incumbent's vocabulary.

---

## 8. Risks

| Risk | Mitigation |
|---|---|
| **Scope creep in P2** — the whole premise of D-5 collapses if 10 weeks becomes 10 months | The NOT-list above is binding. Overrun → cut scope, never extend |
| **CAPI error messages are opaque** | Status aggregation (step 11) is a scoped deliverable, not a nicety. It is also the one place P2 can be better than the free alternatives |
| **Upgrade rabbit hole** | Happy path only. Edge cases go in a file, not the sprint |
| **EC2 bill surprise** | Budget alert day one; teardown is the CLI default |
| **P1 keeps getting deferred** | The exit trigger is the done-test video, not polish and not recruitment |
| **The quirks moat loses months** | Partly bought back by read-only BIOS capture landing in week 4 |
