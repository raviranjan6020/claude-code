# 06 — The Rest of the Field

The five you named are not the whole competitive set. Several of these are *closer* to
what you're proposing to build than Red Hat or Rafay are.

---

## Spectro Cloud (Palette) — your closest competitor. Study hardest.

**Positioning:** "manage any Kubernetes, anywhere, declaratively, including the OS."
Effectively: CAPI-based multi-distro cluster lifecycle + a very strong edge story.

**Core abstraction — Cluster Profile.** A versioned, layered stack:

```
  ┌─────────────────────────┐
  │ Add-ons  (monitoring,   │  ← layer 5..n, each independently versioned
  │           ingress, GPU) │
  ├─────────────────────────┤
  │ CSI      (Rook, Portworx)│
  ├─────────────────────────┤
  │ CNI      (Cilium, Calico)│
  ├─────────────────────────┤
  │ K8s      (PXK/k3s/RKE2/  │
  │           EKS-D, version)│
  ├─────────────────────────┤
  │ OS       (Ubuntu/RHEL/   │  ← THEY OWN THE OS LAYER. Nobody else in the
  │           Kairos image)  │    "any distro" camp does.
  └─────────────────────────┘
```

Profiles are composable (infra profile + add-on profile), inheritable (cluster
overrides a profile), versioned, and **continuously reconciled** — drift is corrected.
A cluster upgrade is "point the cluster at profile v2".

**Edge story (the strong part):**
- **Kairos** (CNCF sandbox, Spectro-sponsored) — a meta-distribution that turns *any*
  base distro (Ubuntu/openSUSE/Alpine/RHEL) into an immutable, A/B-upgradeable,
  container-image-delivered OS with trusted boot (measured boot + TPM-sealed LUKS).
- **EdgeForge** — build a site-specific installer ISO/USB with the cluster profile,
  registration token, and pre-pulled images baked in.
- **Local UI on the device** — a technician at a retail store plugs in a USB, sees a
  browser UI, picks a site ID. No CLI, no engineer on site.
- **Edge Host registration** — appliance phones home, hub allocates it to a cluster.
- Genuinely works air-gapped and over flaky links.

**Other pieces:** Palette VerteX (FIPS 140-2/3, STIG, for US federal), Palette Virtual
Clusters (vCluster-based), Dev Engine (app-level self-service), and CAPI providers for
MAAS, vSphere, Nutanix, OpenStack, all major clouds, and bare metal via their agent.

**Where it hurts:**
- **DC bare metal (BMC-driven) is weaker than its edge story.** Palette's metal path is
  really "boot our installer, it registers" — closer to Elemental/Assisted than to
  Metal3. No serious firmware/BIOS/RAID desired-state. No network fabric.
- SaaS-first (self-hosted exists but is the less-travelled path).
- Pricing is enterprise-tier; not competitive for high-node-count low-margin buyers.
- Closed source above Kairos.

**Steal:** the layered Cluster Profile model (it's the best product abstraction in the
category), EdgeForge, and the local-UI-at-the-site idea.

---

## Sidero Labs (Talos + Omni) — the best engineering, the narrowest scope

**Talos Linux**: immutable, minimal (~80MB), **no SSH, no shell, no package manager,
no systemd** — the entire OS is controlled by a gRPC API and a single YAML machine
config. Kubernetes is the init system's only real job. Atomic A/B upgrades, KSPP
hardening, SecureBoot, encrypted state partitions.

This is the most defensible answer to L11 drift and L12 CVE surface: **there is nothing
to drift, and nothing to patch that isn't the whole image.** CIS/NIST auditors have
almost nothing to complain about.

**Omni**: the management plane. Notable architectural choices:
- **Deliberately rejects Cluster API.** Omni is machine-centric: machines join a pool,
  you allocate them to clusters via **machine classes** and **cluster templates**.
  Sidero's stated view is that CAPI is a mechanism, not a product, and carries too much
  complexity. Worth taking seriously as a counter-argument to our CAPI bet.
- **SideroLink**: every machine establishes a **WireGuard** tunnel to Omni on boot.
  Omni thereby gets a routable path to the Talos API on every node anywhere on earth,
  with no inbound firewall rules and no exposed Talos API. Machines register by booting
  an Omni-generated image containing a join token.
- **KubeSpan**: extends WireGuard mesh between nodes, so a cluster can span sites/
  clouds/NAT transparently.
- **Image Factory**: build a Talos image with chosen **system extensions** (NVIDIA
  drivers, iSCSI, DRBD, etc.) — the answer to "immutable OS but I need drivers".
- **Infrastructure Providers** (2025) — including a bare-metal provider doing PXE +
  IPMI/Redfish power control, plus cloud providers. This is them slowly rebuilding
  Metal3-ish capability under an Omni-native API. (The older `Sidero Metal` CAPI
  provider is deprecated.)
- SaaS or self-hosted; SAML/OIDC, ACLs, audit.

**Where it hurts:** Talos-only (a hard no for anyone needing RHEL/Ubuntu userspace,
agents, or legacy software on the node), thin enterprise governance/multi-tenancy,
no VM story, no L3/L6, small company.

**Steal:** SideroLink's "WireGuard-on-boot from a token-baked image" onboarding — it is
simpler and more robust than a TLS reverse tunnel and gives you *node-level* reach, not
just API-level. Also Image Factory's system-extension model.

---

## Canonical — MAAS is the best standalone metal tool

**MAAS** (Metal as a Service): DHCP + DNS + PXE/HTTP boot + IPAM + machine lifecycle +
commissioning, with a clean REST API and a decent UI.

Machine lifecycle: `New → Commissioning → Ready → Allocated → Deploying → Deployed →
Releasing (+ optional disk erase)`.

Strengths: very wide power-driver support (IPMI, Redfish, AMT, iDRAC, LXD, Proxmox,
vSphere, Hetzner, wedge/OCP switches), **built-in IPAM/DNS/DHCP** (the layer everyone
else makes you supply), declarative network *and storage* layout per machine
(bcache/RAID/LVM/partitions), availability zones, resource pools, tags, composable
hardware via LXD "VM hosts", and curtin/cloud-init for install.

**Watch the license: MAAS is AGPLv3.** For a commercial product that embeds or
network-serves it, get legal advice before building on it. This alone may push us to
Metal3/Tinkerbell.

Above it: **Juju** (model-driven charms — powerful, but a whole cultural commitment
that most teams reject), **Canonical Kubernetes**/Charmed K8s, **Landscape** for OS
patching, **Ubuntu Core** (snap-based immutable) for devices.

**Where it hurts:** Ubuntu-centric, Postgres-backed monolith rather than K8s-native,
Juju is a barrier, and Canonical has relaunched its Kubernetes offering enough times
that enterprises hesitate.

**Steal:** the commissioning model, the storage-layout declaration, and the fact that
**DHCP/DNS/IPAM ownership is part of the product**. Everyone who omits this generates
support tickets forever.

---

## Tinkerbell (CNCF Sandbox) — the light, hackable alternative to Metal3

Components (now consolidated into one `tinkerbell/tinkerbell` repo):
- **Smee** — DHCP + iPXE + HTTP boot + syslog
- **Tootles** (ex-Hegel) — EC2-compatible metadata service
- **Tink** (server/controller/worker) — the workflow engine
- **Rufio** — the BMC controller (IPMI/Redfish power, boot device, via `bmclib`)
- **HookOS** — the in-memory install environment

CRDs: `Hardware`, `Template`, `Workflow`, `Machine`, `Job`, `Task`.

**The model:** a `Template` is a sequence of **container actions** run by HookOS on the
target machine (stream an image to disk, write cloud-init, kexec, etc.). Provisioning is
literally "run these containers on the bare machine". Extremely hackable — want to run
a firmware update? Write a container.

**Used by:** AWS **EKS Anywhere** bare metal (via CAPT, the CAPI Tinkerbell provider),
Equinix Metal (its origin).

**vs Metal3:** Tinkerbell is far simpler to run and reason about, and its workflow model
is better for custom per-site steps. Metal3/Ironic has a vastly wider BMC/hardware
matrix, virtual-media boot, and real firmware/RAID/cleaning. Tinkerbell requires you to
own DHCP on the provisioning network (or do careful proxy-DHCP), which is often
politically impossible in enterprise DCs.

Still **Sandbox** maturity as of 2026 — smaller community than the adoption suggests.

---

## Gardener (SAP) — the best *scale* architecture, ignored by everyone

Model: **Garden** cluster (the API/hub) → **Seed** clusters (regional hosting tier) →
**Shoot** clusters (user clusters).

- A Shoot's **control plane runs as pods in a Seed**, with the Seed providing
  monitoring, logging, autoscaling and etcd backup (via `etcd-druid`) for all its
  shoots.
- **`gardenlet`** is the agent in each Seed — same hub-spoke pull pattern, one level up.
- Everything infra-specific is an **Extension** (a Helm-deployed controller per
  provider/OS/CNI/DNS), so the core stays generic.
- Uses **Machine Controller Manager** (its own, pre-CAPI) for node lifecycle.
- Runs **thousands of clusters** in production at SAP across many providers.

**Why it matters to us:** it is the only open-source project that has actually operated
this shape at enterprise-SaaS scale for years, and the **Garden → Seed → Shoot** tiering
is the same insight as Mirantis's management/regional/child. Two independent teams
converged on it. **That's a strong signal our design should have three tiers, not two.**

**Where it hurts:** steep learning curve, SAP-centric community, bare-metal is not a
first-class citizen, docs assume you're building a cloud.

## Kubermatic (KKP)

Master → Seed → User cluster, user-cluster control planes as pods in seeds. Similar
shape to Gardener, more product-ish, strong in European telco/enterprise/edge.
`KubeOne` for bootstrapping. Worth a look for how they package the same idea
commercially.

---

## Platform9

SaaS (or on-prem) control plane + agents on nodes; long history in "managed K8s on your
infra". Has pivoted hard into **Private Cloud Director** — a KVM-based VMware
replacement — which tells you the same thing Harvester does about where enterprise
budget is moving. Also "Elastic Machine Pool" for cost optimisation.

## Nutanix (NKP)

Nutanix Kubernetes Platform — built on the D2iQ acquisition (Konvoy/Kommander). Strong
where Nutanix AHV already exists; a credible VMware-replacement bundle. Kommander's
multi-cluster model is another hub-spoke reference.

## Hyperscaler "anywhere" products

- **EKS Anywhere** (Tinkerbell for bare metal, CAPI-based, free software + paid support).
- **Google Distributed Cloud** (ex-Anthos bare metal; also an air-gapped appliance
  variant for sovereign/defense — worth studying for the sovereign motion).
- **Azure Arc** — the purest "hub-spoke by agent" play: install Arc agents on *any*
  cluster, it appears in Azure Resource Manager, and you get GitOps (Flux extension),
  Policy, Monitor, and RBAC from Azure. **AKS on Azure Local** (ex-Azure Stack HCI) adds
  the on-prem stack. Arc's genius is making on-prem clusters into first-class *cloud*
  resources — billing, IAM, and tooling all unify.
- **AWS Outposts** — the "we ship you the rack" extreme.

## HPC / AI-specific

- **NVIDIA Base Command Manager** (ex-Bright Cluster Manager) — decades-old HPC cluster
  provisioning (stateless node images, Slurm, monitoring), now the DGX SuperPOD default,
  plus **NVIDIA Mission Control** for the full AI-factory lifecycle. If your ICP is
  AI/GPU, **this is the incumbent to beat in the DC**, not Red Hat.
- **Warewulf** / **xCAT** / **OpenHPC** — stateless PXE provisioning for HPC; thousands
  of nodes, ramdisk images, zero day-2 drift. Old, unglamorous, extremely effective.
- **Run:ai** (NVIDIA), **Kueue**, **Volcano**, **HAMi** — GPU scheduling/quota layers.
- **SLURM-on-K8s** convergence (Slinky, etc.) — the real fight in AI clusters is that
  researchers want Slurm and platform teams want Kubernetes.

## Multi-cluster orchestration (the layer above the hub)

- **Karmada** (CNCF incubating) — propagation policies, cross-cluster scheduling,
  cluster-level failover. Aggregated API so `kubectl` works fleet-wide.
- **KubeFleet** (Microsoft, CNCF sandbox) — similar, from the AKS Fleet work.
- **Liqo** — cluster "peering"; a remote cluster appears as a virtual node in yours.
- **Clusternet**, **Admiralty**, **Submariner** (cross-cluster networking).

Relevant but **not our first problem**. Customers rarely buy cross-cluster workload
scheduling before they can reliably build and upgrade a single cluster fleet.

---

## Bare-metal provisioning engines, compared

| | **Metal3 / Ironic** | **Tinkerbell** | **MAAS** | **Omni provider** | **Digital Rebar** |
|---|---|---|---|---|---|
| License | Apache 2 | Apache 2 | **AGPLv3** | BSL/commercial | commercial |
| K8s-native CRDs | yes | yes | no (REST/Postgres) | no (Omni API) | no |
| BMC matrix | **widest** | good (bmclib) | wide | IPMI/Redfish | **widest+** |
| Virtual media boot | **yes** | no | limited | limited | yes |
| Needs to own DHCP | **no** (vmedia) | yes | yes (it *is* DHCP) | yes (PXE) | yes |
| Firmware/BIOS/RAID | yes (declarative) | DIY via actions | weak | no | **best** |
| Disk cleaning/erase | **yes** | DIY | yes | yes | yes |
| IPAM/DNS included | no (separate) | no | **yes** | no | yes |
| CAPI provider | CAPM3 | CAPT | CAPMAAS | n/a (rejects CAPI) | plugin |
| Operational weight | **heavy** | light | medium | light | medium |
| Hackability | low | **high** | medium | low | medium |

**Practical conclusion for our design:** Metal3 for DC/BMC-driven, plus a
registration-first path (Elemental/Assisted/Omni-style ISO that phones home) for edge
and BMC-less sites. Those two paths cover ~95% of real deployments. Tinkerbell stays on
the table as a lighter alternative if Ironic's operational weight becomes a problem.
