# 01 — Red Hat: OpenShift + RHACM + Metal3 + ZTP

**One-line summary:** the most complete, most opinionated, most expensive answer.
Everything is a CRD, everything is GitOps, everything assumes OpenShift.

Red Hat is the benchmark. If you can explain why a customer should *not* buy OpenShift,
you have a business. Study this stack hardest — much of it is Apache-2 licensed and
reusable.

---

## 1. The layer cake

```
                        ┌──────────────────────────────────────────┐
                        │  HUB CLUSTER (OpenShift)                 │
                        │                                          │
  Git repo  ──Argo CD──▶│  RHACM  (= Open Cluster Management +)    │
  (SiteConfig,          │   ├─ MCE (multicluster engine)           │
   PolicyGenerator)     │   │   ├─ Hive         (ClusterDeployment)│
                        │   │   ├─ Assisted Svc (InfraEnv/Agent)   │
                        │   │   ├─ Metal3/BMO   (BareMetalHost)    │
                        │   │   ├─ CAPI + CAPM3                    │
                        │   │   └─ HyperShift   (HostedCluster)    │
                        │   ├─ Governance  (Policy/Placement)      │
                        │   ├─ Observability (Thanos)              │
                        │   ├─ Search, App lifecycle               │
                        │   └─ TALM (ClusterGroupUpgrade)          │
                        └────────────┬─────────────────────────────┘
                                     │ klusterlet pulls ManifestWork (mTLS, outbound)
          ┌──────────────────────────┼──────────────────────────┐
          ▼                          ▼                          ▼
   SNO @ cell site          3-node @ regional DC         Hosted CP cluster
   (RHCOS, RAN DU profile)  (RHCOS, ODF, Virt)           (CP pods on hub)
```

---

## 2. Layer-by-layer

### L1–L4: Metal3 (`metal3.io`) — Apache 2.0, the crown jewel

Metal3 is Red Hat's bare-metal engine and is **directly reusable by us**.

Components:
- **baremetal-operator (BMO)** — the Kubernetes controller. Reconciles `BareMetalHost`.
- **Ironic** — inherited from OpenStack. The actual provisioning state machine:
  power, boot device, deploy, clean, inspect. Talks Redfish/IPMI/iDRAC/iLO/iRMC/
  redfish-virtualmedia/idrac-virtualmedia and ~15 more drivers. **The hardware
  compatibility matrix here is the widest in open source — years of work.**
- **Ironic Python Agent (IPA)** — ramdisk that runs on the node during
  inspection/cleaning/deploy. Writes the image, reports hardware.
- **Ironic Inspector**, **httpd image cache**, **dnsmasq** (optional; not needed on the
  virtual-media path).

Key CRDs:

| CRD | Purpose |
|---|---|
| `BareMetalHost` | The machine. BMC address+creds, boot MAC, image, userData, state |
| `HostFirmwareSettings` | Declarative BIOS attributes (desired vs current) |
| `FirmwareSchema` | Vendor-specific attribute schema discovered from the BMC |
| `HostFirmwareComponents` | Firmware versions (BMC/BIOS) + update targets |
| `HostUpdatePolicy` | Whether live firmware/settings changes are allowed |
| `PreprovisioningImage` | The IPA ramdisk/ISO built per host |
| `DataImage` | Attach an arbitrary ISO to the host via virtual media |
| `BMCEventSubscription` | Redfish event subscription (async alerts) |

`BareMetalHost` state machine:
`registering → inspecting → preparing (clean/RAID/firmware) → available → provisioning
→ provisioned → deprovisioning → deleting`

CAPI integration — **cluster-api-provider-metal3 (CAPM3)**:
`Metal3Cluster`, `Metal3MachineTemplate`, `Metal3Machine`, `Metal3DataTemplate`,
`Metal3Data`, `Metal3DataClaim`, plus the `ip-address-manager` (`IPPool`, `IPClaim`,
`IPAddress`) which does static IPAM for metal — genuinely useful, most edge sites
don't want DHCP for prod interfaces.

**What Metal3 gives you that nothing else does:** Redfish **virtual media** deploy
(no DHCP/TFTP ownership — mount the IPA ISO over HTTPS from the BMC), plus declarative
firmware/BIOS/RAID, plus automatic cleaning (secure erase between tenants — essential
for multi-tenant metal-as-a-service).

**What it costs you:** Ironic is an OpenStack-era Python service with a conductor,
a database, and a lot of config. Operationally heavy. The docs assume OpenStack or
OpenShift context. Debugging a stuck `BareMetalHost` means reading Ironic node state.

### L4 (OS): RHCOS → bootc / RHEL image mode

RHCOS is an immutable, `rpm-ostree`-based OS. No `yum install` on a node — you change
the OS by changing a `MachineConfig`, and the **Machine Config Operator (MCO)** does a
rolling, ordered, drain-aware reboot per `MachineConfigPool`. Ignition (not cloud-init)
does first-boot config.

The strategic move is **bootc / "RHEL image mode"**: the OS becomes a container image
you build with a Containerfile, push to a registry, sign with sigstore, and nodes
`bootc upgrade` to pull. Same registry, same mirroring, same signing, same air-gap
tooling as your workloads. **This is the most important architectural trend in the
whole space and we should build on it.**

For telco: "on-cluster layering" lets a cluster build its own OS image with extra
drivers (GPU, DPU) as a `MachineOSConfig` — solving the perennial "immutable OS but I
need an out-of-tree kernel module" problem.

### L7: three installers, not one

OpenShift deliberately does **not** use CAPI for day-1 bare metal. Instead:

1. **IPI (Installer-Provisioned Infrastructure) bare metal** — the installer runs a
   bootstrap VM, talks to BMCs via Ironic, provisions everything. Needs the installer
   host to reach the BMC network.
2. **Assisted Installer** (`openshift/assisted-service`, Apache 2) — **this is the
   clever one.** You define an `InfraEnv`, it generates a **discovery ISO**. You boot
   any machine from it (virtual media, USB, PXE — your choice). The agent phones home,
   creates an `Agent` CR, runs preflight validations (DNS, NTP, connectivity, disk,
   CPU flags), and then installs. It works when you *cannot* reach the BMC, which is
   the normal case at customer edge sites. Backed by `ClusterDeployment` and
   `AgentClusterInstall` (Hive CRDs).
3. **Agent-based Installer** — the same engine packaged fully offline as a single ISO,
   no hub required. For true air-gap first-cluster bootstrap.

**Hive** (Apache 2) is the underlying cluster-provisioning operator:
`ClusterDeployment`, `ClusterPool` (pre-warmed clusters, claim on demand — a great
idea for CI/dev fleets), `ClusterImageSet`, `SyncSet`/`SelectorSyncSet`, `MachinePool`.

### L10: RHACM = Open Cluster Management (OCM) + Red Hat extras

The open core is **`open-cluster-management.io`** (Apache 2, CNCF sandbox), upstream
of RHACM. Architecture:

- **Hub**: `ClusterManager` deploys registration-controller, work-controller,
  placement-controller, addon-manager.
- **Spoke**: **`klusterlet`** = `registration-agent` (identity, heartbeat, status) +
  `work-agent` (pulls `ManifestWork`, applies, reports).
- **Registration handshake**: spoke agent creates a CSR on the hub; a human or
  auto-approver approves; hub issues a client cert; **mTLS, always outbound from the
  spoke**. Each managed cluster gets a **dedicated namespace on the hub** — the natural
  isolation and quota boundary.
- **`ManifestWork`**: a wrapper around a list of raw manifests to apply on a spoke,
  with `feedbackRules` to pull specific status fields back to the hub. The hub *never*
  stores spoke kubeconfigs and never initiates a connection.
- **`Placement` / `PlacementDecision`**: declarative "which clusters" — predicates on
  labels/claims, `spreadPolicy` across topology, prioritizers (resource, steady state).
  `ManagedClusterSet` + `ManagedClusterSetBinding` gives the tenancy boundary.
- **addon-framework**: `ClusterManagementAddOn` + `ManagedClusterAddOn` — a standard
  way to ship an agent to every spoke with its own hub-issued identity. Everything
  else (observability, policy, search, submariner) is an addon.

**Verdict: OCM is the single best open-source answer to "how do I build a hub".**
Pull-based, no stored credentials, per-cluster namespace isolation, scales to
thousands. If we build a hub, starting from OCM saves a year.

RHACM adds on top: Governance (`Policy`, `PlacementBinding`, config/Gatekeeper/Kyverno
policies with `inform` vs `enforce` modes), multicluster observability (Thanos on hub,
`metrics-collector` on spokes with allowlists to control cardinality), Search
(a graph of all fleet objects), Application lifecycle, and Submariner for cross-cluster
pod networking.

### ZTP: how 1000 sites actually get deployed

The flow:

```
Git repo
 ├─ site-configs/     ClusterInstance CRs (per-site: BMC, MACs, IPs, VLANs, disk)
 └─ policies/         PolicyGenerator CRs (per-profile: RAN DU, PTP, SR-IOV, tuning)
         │
      Argo CD (ApplicationSet with a cluster generator)
         │
         ▼
  SiteConfig Operator ──renders──▶ ClusterDeployment + AgentClusterInstall
                                   + InfraEnv + BareMetalHost + NMStateConfig
         │
         ▼
  Assisted Service ──discovery ISO via virtual media──▶ server boots, installs
         │
         ▼
  Cluster joins hub (klusterlet) ──▶ Policies apply ──▶ compliant
         │
         ▼
  TALM (ClusterGroupUpgrade) ──batched canary rollout of day-2 changes
```

Key evolution: the old **`SiteConfig` kustomize plugin** was replaced by the
**SiteConfig Operator** with a `ClusterInstance` CR + pluggable **installation
templates**. This decouples *what the site is* (`ClusterInstance`) from *how it gets
installed* (template: assisted-installer / image-based-install / IPI). That separation
is exactly the right abstraction and we should steal it.

**TALM** (`ClusterGroupUpgrade`) is the day-2 rollout engine: select clusters, define
batch size and canaries, pre-cache images at the site *before* the maintenance window,
take a backup, apply, verify, proceed or abort. **Nobody else has a day-2 rollout
primitive this good.** It's the difference between "GitOps applied it" and "1000 cell
sites upgraded without a truck roll."

**Image-Based Install/Upgrade (IBI/IBU)** for single-node telco: bake a "seed image"
from a fully-configured reference SNO, ship it, and a site install becomes an ostree
pivot measured in minutes instead of an hour. Upgrades become an A/B image swap with
instant rollback.

### HyperShift / Hosted Control Planes

`HostedCluster` + `NodePool`. The control plane (apiserver, etcd, controllers) runs as
**pods on the hub**; worker nodes join over the network via konnectivity. On bare metal
this uses the "Agent" platform — workers come from the same `Agent`/`InfraEnv` pool.

Economics: 100 small clusters × 3 control plane nodes = 300 servers of pure overhead.
With HCP that's ~0 dedicated servers and control planes upgrade as a pod rollout.
**But**: workers now depend on WAN reachability to the hub for the API server. Fine for
a regional DC, **wrong for a disconnected far-edge site**.

---

## 3. Telco specifics (the reason Red Hat wins RAN)

`PerformanceProfile` (Node Tuning Operator) → CPU partitioning (isolated vs reserved
cores), hugepages, real-time kernel, IRQ affinity; `SriovNetworkNodePolicy` /
`SriovNetwork`; PTP operator (grandmaster/boundary/ordinary clock for 5G timing);
NUMA Resources Operator + topology-aware scheduling; workload partitioning so the
platform itself is pinned to 2 cores. None of this is glamorous and all of it is
required to win a telco RFP.

---

## 4. Business model

- Priced per **core-pair** or per **socket**, annual subscription. Roughly
  **$3–5k+/core-pair/yr list** depending on tier; "OpenShift Platform Plus" bundles
  ACM + ACS (StackRox) + Quay + ODF.
- Self-managed and managed (ROSA/ARO) variants.
- Effectively: **5–15× the price of a Rancher-class stack**. On a 1000-node fleet that
  is a multi-million-dollar annual line item.

## 5. Where it hurts

1. **Price.** The most common reason a deal is lost. Entire market segments (edge with
   thousands of tiny sites, GPU neoclouds with thin margins, mid-market, most of
   APAC/India) simply cannot pay it.
2. **All-or-nothing.** You get OpenShift or you get nothing. Can't manage an RKE2
   cluster, can't manage a vanilla kubeadm cluster, can't manage someone else's EKS as
   a first-class citizen.
3. **Resource footprint.** A "small" OpenShift cluster is not small. SNO wants 8+ cores
   and 16-32GB before your workload exists.
4. **Complexity.** The ZTP stack (Argo + SiteConfig Operator + Assisted + Metal3 + Hive
   + Policies + TALM) requires a dedicated team to run. Time-to-first-cluster for a new
   customer is measured in weeks.
5. **Rigidity of the OS.** MachineConfig is powerful but every change is a rolling
   reboot of a pool. Customers with exotic drivers fight this constantly.
6. **Bare metal beyond OpenShift.** They have no answer for "provision me 200 GPU
   servers running plain Ubuntu for a Slurm cluster."

## 6. What we steal

- **Metal3 / Ironic** wholesale (Apache 2). The hardware matrix is the moat we don't
  have to build.
- **OCM** as the hub substrate (Apache 2).
- **Assisted-Installer's model**: "if you can't reach the BMC, ship an ISO that phones
  home." Critical for edge.
- **`ClusterInstance` + installation templates** separation of concerns.
- **TALM's `ClusterGroupUpgrade`** semantics: canary + batch + pre-cache + backup +
  abort. Re-implement this for any distro and you have a feature nobody else has.
- **bootc** as the OS distribution mechanism.
